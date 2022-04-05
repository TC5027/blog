# Optimizing deserialization with an alternative to Vec

On considère des données de taille dynamique, par exemple un vecteur d'octets.
On pose l'alias suivant:

```rust
type Bytes = Vec<u8>;
```

On veut écrire ces données sur disque puis les recharger en mémoire

Pour les écrire sur disque (les sérialiser) on peut prendre l'approche suivante:

* un header qui contient la taille totale de la séquence et le nombre de vecteurs représentés
* les tailles des vecteurs, écrites successivement
* les données même, toutes concaténées

La sérialisation peut donc se faire de cette manière:

```rust
fn serialize(set_of_bytes: Vec<Bytes>) -> Vec<u8> {
    let nb_sets = set_of_bytes.len();
    let total_length = std::mem::size_of::<usize>()
        + std::mem::size_of::<usize>() * nb_sets
        + set_of_bytes.iter().map(|bytes| bytes.len()).sum::<usize>();
    // size to write nb_sets
    // + size to write length fields
    // + size to write all content

    let mut result: Vec<u8> = Vec::with_capacity(std::mem::size_of::<usize>() + total_length);
    result.append(&mut Vec::from(total_length.to_ne_bytes()));
    result.append(&mut Vec::from(nb_sets.to_ne_bytes()));

    for bytes in &set_of_bytes {
        result.append(&mut Vec::from(bytes.len().to_ne_bytes()));
    }
    for mut bytes in set_of_bytes {
        result.append(&mut bytes);
    }

    result
}
```

Et la désérialisation comme ceci:

```rust
fn deserialize1<R>(mut storage: R) -> Vec<Bytes>
where
    R: Read,
{
    // we get the total_length to know until where we should read
    let mut a_usize = [0; std::mem::size_of::<usize>()];
    storage.read_exact(&mut a_usize).unwrap();
    let total_length = usize::from_ne_bytes(a_usize);
    // we get all the data
    let mut full_data = vec![0; total_length];
    storage.read_exact(&mut full_data).unwrap();
    // we get the number of sets
    let nb_sets = usize::from_ne_bytes(full_data[0..8].try_into().unwrap());
    // we split full_data between the sizes and the contents
    let usize_size = std::mem::size_of::<usize>();
    let mut contents = full_data.split_off(usize_size + usize_size * nb_sets);
    // we reform the sizes from bytes
    let mut sizes: Vec<usize> = Vec::with_capacity(nb_sets);
    for i in 0..nb_sets {
        sizes.push(usize::from_ne_bytes(
            full_data[usize_size * (i + 1)..usize_size * (i + 2)]
                .try_into()
                .unwrap(),
        ));
    }
    let mut result = Vec::with_capacity(nb_sets);
    for size in sizes {
        let remaining_contents = contents.split_off(size);
        result.push(contents);
        contents = remaining_contents;
    }

    result
}
```

On peut vérifier nos fonctions avec le ```main``` suivant:

```rust
fn main() {
    let bytes1: Bytes = vec![1,2,3];
    let bytes2: Bytes = vec![4,5];
    let serialized = serialize(vec![bytes1, bytes2]);
    let mut storage: File = OpenOptions::new().append(true).read(true).create(true).open("storage.strg").unwrap();
    storage.write_all(&serialized).unwrap();
    storage.seek(SeekFrom::Start(0)).unwrap();
    assert_eq!(vec![vec![1,2,3],vec![4,5]], deserialize1(storage));
}
```

Si on considère un ```main``` plus "imposant" comme celui-ci par exemple:

```rust
fn main() {
    let set_of_bytes = (0..10_000).into_iter().map(|_| vec![0; 50]).collect();
    let serialized = serialize(set_of_bytes);
    let mut storage: File = OpenOptions::new()
        .append(true)
        .read(true)
        .create(true)
        .open("storage.strg")
        .unwrap();
    storage.write_all(&serialized).unwrap();
    storage.seek(SeekFrom::Start(0)).unwrap();
    assert_eq!(10_000, deserialize1(storage).len());
}
```

Pour investiguer et mieux comprendre l'éxécution de mon programme je peux utiliser un flamegraph qui sert à visualiser les stack traces. Pour le lancer je fais préalablement les commandes suivantes :
```cmd
echo -1 | sudo tee /proc/sys/kernel/perf_event_paranoid
echo 0 | sudo tee /proc/sys/kernel/kptr_restrict
```

et ensuite je fais 
```cmd 
cargo flamegraph
```
qui me produit un joli .svg :D .

![ouch](https://github.com/TC5027/blog/blob/master/pngs/bad.svg)

Oula... Pour l'analyse, on voit que notre ```main``` passe beaucoup de temps au niveau de ```deserialize1``` et plus précisément au niveau de la méthode ```split_off``` de ```Vec```. On peut regarder son [code source](https://doc.rust-lang.org/src/alloc/vec/mod.rs.html#1928-1930) et on voit que cette méthode provoque la création d'un nouveau vecteur alloué ( voir [1](https://doc.rust-lang.org/beta/src/alloc/raw_vec.rs.html#132) et [2](https://doc.rust-lang.org/beta/src/alloc/raw_vec.rs.html#170) ). En y réfléchissant, c'est un peu idiot de devoir faire une nouvelle allocation car tout est déja là vu qu'on a tout chargé depuis le disque ! On a donc une consommation mémoire inutilement importante qui provoque surement les page fault que l'on voit dans le flamegraph et qui ont une part non négligeable !
On voit donc qu'il se passe plein de choses dans le flamegraph, mais surement TROP considéré ce que l'on fait ? ```Vec``` ne semble pas etre la bonne piste pour cette désérialisation.

On propose ```SliceableBytes```, inspiré de ```Vec``` mais basé sur ```CustomAlloc```.

```SliceableBytes``` comprend donc, à la manière de ```raw_vec```, un champ pointant le début des données, une taille de plage et un allocateur pour gérer la mémoire.

```rust
#[derive(Debug)]
struct SliceableBytes {
    start: *const u8,
    size: usize,
    allocator: Rc<RefCell<CustomAlloc>>,
}
```
Ce ```SliceableBytes``` a à sa création une instance de ```CustomAlloc``` qui lui est attribuée. Ensuite, les ```SliceableBytes``` issus de ce premier, qu'on obtiendra à partir d'appels à ```split```, partagerons avec le premier cette instance de ```CustomAlloc``` (pour cela qu'on wrap ```CustomAlloc``` dans un ```Rc<Refcell<>>``` afin de le partager et de rendre les modifications faites par un ```SliceableBytes``` visible pour tous). ```CustomAlloc``` lui, n'est dans le fond qu'un compteur de ```SliceableBytes```, gérant une zone mémoire déja allouée par l'allocateur ```Global```.
```rust
#[derive(Debug)]
struct CustomAlloc {
    playground: Vec<u8>,
    counter: usize,
}
```
Avec un call à ```alloc``` on met le compteur à 1 et lorsque le compteur est à 0 et qu'on a donc plus d'utilisation, on drop tout.
On a donc un usage un peu batard de ```CustomAlloc```, ce n'est pas un allocateur global mais plutot un allocateur pour une plage qu'on a reçu de l'allocateur ```Global``` et dont on assure la gestion avec ```CustomAlloc```.
Pour construire un ```SliceableBytes```, étant donné notre usage, on prend en entrée un vecteur d'u8, chargé depuis le disque, qu'on veut pouvoir découper. On récupère la taille du vecteur pour l'argument size et on passe ensuite le vecteur au constructeur de ```CustomAlloc```. Le vecteur d'u8, alloué par l'allocateur ```Global``` va maintenant etre géré par ```CustomAlloc``` qui s'en sert un peu comme d'un bac à sable. On a finalement un call à ```alloc``` qui nous donne le start et voila.
```rust
impl SliceableBytes {
    pub fn new(to_be_sliced: Vec<u8>) -> Self {
        let size = to_be_sliced.len();    
        let mut allocator = CustomAlloc::new(to_be_sliced);
        let start = allocator.alloc();
        Self {
            start,
            size,
            allocator: Rc::from(RefCell::from(allocator)),
        }
    }
}
```
Au niveau de ```CustomAlloc``` on prend simplement le controle du vecteur et on initialise le compteur à 0. ```alloc``` qui indique qu'on s'en sert met lui le compteur à 1 et renvoie le début de la plage.
```rust
impl CustomAlloc {
    pub fn new(playground: Vec<u8>) -> Self { 
        Self {
            playground,
            counter: 0,
        }
    }

    pub fn alloc(&mut self) -> *const u8 {
        self.counter = 1;
        self.playground.as_ptr()
    }
}
```
La seconde méthode qui fait tout l'intéret de ```SliceableBytes``` est ```split```, qui en retourne une nouvelle instance.

On effectue un test sur la faisabilité de cette coupe étant donné la taille de notre plage et on appelle ensuite la méthode ```split``` de notre allocateur. On met à jour notre size et on renvoie un nouveau ```SliceableBytes``` correspondant à la droite de la coupe (l'original étant maintenant la gauche de celle-ci).
```rust
pub fn split(&mut self, index: usize) -> SliceableBytes {
    assert!(index <= self.size);
    self.allocator.borrow_mut().split();
    let size = self.size - index;
    self.size = index;
    SliceableBytes {
        start: unsafe { self.start.add(index) },
        size,
        allocator: self.allocator.clone(),
    }
}
```
Au niveau de ```CustomAlloc```, ```split``` n'est qu'une incrémentation du compteur, on a une instance en plus qui utilise le playground.
```rust
pub fn split(&mut self) {
    self.counter += 1;
}
```
Enfin on implémente ```Drop``` pour ```SliceableBytes``` en appelant le ```dealloc``` de ```CustomAlloc```.
```rust
impl Drop for SliceableBytes {
    fn drop(&mut self) {
        self.allocator.borrow_mut().dealloc();
    }
}
```
et ```dealloc``` de ```CustomAlloc``` fait un test sur son compteur et s'il n'y a plus d'usage ```drop``` également
```rust
pub fn dealloc(&mut self) {
    self.counter -= 1;
    if self.counter == 0 {
        drop(self)
    }
}
```
On peut maintenant revoir notre code de deserialize en utilisant cette nouvelle structure ```SliceableBytes```

```rust
fn deserialize2<R>(mut storage: R) -> Vec<SliceableBytes>
where
    R: Read,
{
    // we get the total_length to know until where we should read
    let mut a_usize = [0; std::mem::size_of::<usize>()];
    storage.read_exact(&mut a_usize).unwrap();
    let total_length = usize::from_ne_bytes(a_usize);
    // we get all the data
    let mut full_data = vec![0; total_length];
    storage.read_exact(&mut full_data).unwrap();
    // we get the number of sets
    let nb_sets = usize::from_ne_bytes(full_data[0..8].try_into().unwrap());
    // we split full_data between the sizes and the contents
    let usize_size = std::mem::size_of::<usize>();
    let contents = full_data.split_off(usize_size + usize_size * nb_sets);
    // we are going to slice contents
    let mut sliceable_contents = SliceableBytes::new(contents);
    // we reform the sizes from bytes
    let mut sizes: Vec<usize> = Vec::with_capacity(nb_sets);
    for i in 0..nb_sets {
        sizes.push(usize::from_ne_bytes(
            full_data[usize_size * (i + 1)..usize_size * (i + 2)]
                .try_into()
                .unwrap(),
        ));
    }
    let mut result = Vec::with_capacity(nb_sets);
    // ONLY CHANGE
    for size in sizes {
        let remaining_contents = sliceable_contents.split(size);
        result.push(sliceable_contents);
        sliceable_contents = remaining_contents;
    }

    result
}
```

En rejouant le même ```main``` qu'avant, on voit comme preuve de notre réussite que le flamegraph s'est considérablement simplifié, on passe de 700 samples pour all à 7.

![mieux](https://github.com/TC5027/blog/blob/master/pngs/good.svg)
