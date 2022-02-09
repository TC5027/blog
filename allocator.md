# Design d'un allocateur personnalisé

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
fn serialize(set_of_bytes : Vec<Bytes>) -> Vec<u8> {
    let nb_sets = set_of_bytes.len();
    let total_length = std::mem::size_of::<usize>()
        + std::mem::size_of::<usize>()*nb_sets
        + set_of_bytes.iter().map(|bytes| bytes.len()).sum::<usize>();
    // size to write nb_sets
    // + size to write length fields
    // + size to write all content
    
    let mut result: Vec<u8> = Vec::with_capacity(
        std::mem::size_of::<usize>()+
        total_length
    );
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
fn deserialize<R>(mut storage: R) -> Vec<Bytes>
    where R: Read
{
    // we get the total_length to know until where we should read
    let mut a_usize = [0;std::mem::size_of::<usize>()];
    storage.read_exact(&mut a_usize).unwrap();
    let total_length = usize::from_ne_bytes(a_usize);
    // we get all the data
    let mut full_data = vec![0;total_length];
    storage.read_exact(&mut full_data).unwrap();
    // we get the number of sets
    let nb_sets = usize::from_ne_bytes(full_data[0..8].try_into().unwrap());

    // we split full_data between the sizes and the contents
    let usize_size = std::mem::size_of::<usize>();
    let mut contents = full_data.split_off(usize_size+usize_size*nb_sets);

    // we reform the sizes from bytes
    let mut sizes: Vec<usize> = Vec::with_capacity(nb_sets);
    for i in 0..nb_sets {
        sizes.push(usize::from_ne_bytes(
            full_data[usize_size*(i+1)..usize_size*(i+2)]
            .try_into().unwrap())
        );
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

    assert_eq!(vec![vec![1,2,3],vec![4,5]], deserialize(storage));
}
```

Quand on analyse l'éxécution de ce code avec un main plus "imposant" comme celui-ci par exemple:

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

    assert_eq!(10_000, deserialize(storage).len());
}
```

à l'aide de Windows Performance Recorder et Windows Performance Analyzer, on voit un nombre très conséquent d'allocations durant l'éxécution ![figurebad](https://github.com/TC5027/blog/blob/master/pngs/bad.png)

La méthode ```split_off``` qu'on utilise dans la dernière boucle de ```deserialize``` est responsable de tout ça. En effet quand on regarde son [code source](https://doc.rust-lang.org/src/alloc/vec/mod.rs.html#1928-1930) on voit que cela provoque la création d'un nouveau vecteur alloué quelquepart ( voir [1](https://doc.rust-lang.org/beta/src/alloc/raw_vec.rs.html#132) et [2](https://doc.rust-lang.org/beta/src/alloc/raw_vec.rs.html#170) ).
En y réfléchissant, c'est un peu idiot de devoir faire une nouvelle allocation car tout est déja là vu qu'on a tout chargé depuis le disque !

On aimerait donc pouvoir avoir un allocateur à qui l'on peut dire : "j'ai ce bout de mémoire que tu m'as donné, maintenant fait comme si c'était 2 allocations contigues"

Faire simplement un allocateur répondant à ce besoin ne suffirait pas car ```Vec``` comme on l'a vu fait automatiquement un call alloc via sa méthode ```split_off```. On va donc faire un allocateur + une structure basée dessus, minimalistes, démontrant la faisabilité de ce qu'on cherche à faire.

Qu'est ce qu'un allocateur ? Son but est de gérer la mémoire et son attribution, il doit réquisitionner une région non occupée lors d'un call à ```alloc``` et la retourner avec un call à ```dealloc```.

!TODO explications et vérif avec valgrind

```rust
#[derive(Debug)]
struct CustomAlloc {
    start: *const u8,
    size: usize,
    counter: usize,
}

impl CustomAlloc {
    pub fn new(playground: Vec<u8>) -> Self {
        let size = playground.capacity();
        let start = playground.as_ptr();
        Self {
            start,
            size,
            counter: 0,
        }
    }

    pub fn alloc(&mut self) -> *const u8 {
        self.counter = 1;
        self.start
    }

    pub fn dealloc(&mut self) {
        self.counter -= 1;
        if self.counter == 0 {
            drop(self)
        }
    }

    pub fn split(&mut self) {
        self.counter += 1;
    }
}

impl Drop for CustomAlloc {
    fn drop(&mut self) {
        unsafe { drop(Vec::from_raw_parts(self.start as *mut u8, 0, self.size)) };
    }
}
```

!TODO explications

```rust
#[derive(Debug)]
struct SliceableBytes {
    start: *const u8,
    size: usize,
    allocator: Rc<RefCell<CustomAlloc>>,
}

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
}

impl Drop for SliceableBytes {
    fn drop(&mut self) {
        self.allocator.borrow_mut().dealloc();
    }
}
```

On peut maintenant revoir notre code de deserialize en utilisant cette nouvelle structure ```SliceableBytes```
```rust
fn deserialize<R>(mut storage: R) -> Vec<SliceableBytes>
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
En rejouant le même ```main``` qu'avant on voit que les nombreux calls à alloc ont disparu 
![figuregood](https://github.com/TC5027/blog/blob/master/pngs/good.png)et le temps d'éxécution est diviser par plus de 10 !
!TODO détailler

