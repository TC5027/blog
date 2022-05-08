# Hilbert curve for fast matrix multiplication (single thread)

## Matrix multiplication

Cet article va se concentrer sur la multiplication de matrices (avec une taille en puissance de 2 pour rendre les choses plus simples) et comment en améliorer les performances.

Pour commencer on déclare une structure de matrice
```rust
#[derive(Debug, PartialEq)]
pub struct Matrix<T> {
    pub width: usize,
    pub height: usize,
    data: Vec<T>,
}

impl<T> Matrix<T> {
    pub fn new(width: usize, height: usize, data: Vec<T>) -> Self {
        Matrix {
            width,
            height,
            data,
        }
    }
 
    fn new_uninit(width: usize, height: usize) -> Self {
        let mut data = Vec::with_capacity(width * height);
        unsafe { data.set_len(width * height) };
        Matrix {
            width,
            height,
            data,
        }
    }
}
```

On implémente aussi les traits ```Index``` et ```IndexMut``` pour pouvoir accéder à ses données
```rust
impl<T> Index<(usize, usize)> for Matrix<T> {
    type Output = T;
    fn index(&self, index: (usize, usize)) -> &T {
        &self.data[self.width * index.0 + index.1]
    }
}
 
impl<T> IndexMut<(usize, usize)> for Matrix<T> {
    fn index_mut(&mut self, index: (usize, usize)) -> &mut T {
        &mut self.data[self.width * index.0 + index.1]
    }
}
```

On peut désormais écrire une fonction pour la multiplication, en partant de la définition mathématiques classique
```rust
pub fn multiply<T>(a: &Matrix<T>, b: &Matrix<T>) -> Matrix<T>
where
    T: Sum + AddAssign<T> + Mul<T, Output = T> + Copy + Default,
{
    assert_eq!(a.width, b.height);
    let mut c = Matrix::new_uninit(b.width, a.height);

    for i in 0..c.height {
        for j in 0..c.width {
            c[(i, j)] = (0..a.width).into_iter().map(|k| a[(i,k)]*b[(k,j)]).sum();
        }
    }
    c
}
```

Pour tester on pose le ```main``` suivant
```rust
fn main() {
    let half_log_grid_size = 10;
    let log_grid_size = 2*half_log_grid_size;
    let size = 1 << half_log_grid_size;

    // classic
    let a = Matrix::new(size, size, vec![1; size * size]);
    let b = Matrix::new(size, size, vec![1; size * size]);
    assert_eq!(size, multiply(&a, &b).width);
}
```

Avec la commande ```time``` (après avoir build en mode release) on obtient un résultat de **0m4,375s**
Un des points clés pour la performance c'est l'utilisation du cache c'est à dire parcourir les données dans un ordre qui permet d'éviter d'avoir un grand nombre de cache misses. On peut mesurer cela avec ```valgrind --tool=cachegrind```, qui donne pour notre programme
**D1  miss rate:           50.0% (         50.0%     +      73.1%  )**. La première valeur dans la parenthèse représente le pourcentage de lectures qui tentaient d'accéder à des données qui ne sont pas en cache, et le deuxième pourcentage va pour les écritures. La valeur avant la parenthèse représente le pourcentage d'accès (lecture + écriture) qui étaient vers des données non présentes dans le cache.
On voit donc que c'est loin d'etre satisfaisant et cela n'est pas étonnant. On multiplie des lignes par des colonnes. Les données de la colonne ne sont pas contigues dans la mémoire de par la représentation en ligne pour ```Matrix``` et donc le cache en souffre ce qui explique nos résultats.

## Transpose

Pour mieux parcourir les données on peut d'abord transposer la seconde matrice et ainsi ne parcourir que des lignes !

$c_{i,j} = \sum_{k}a_{i,k}b_{k,j} = \sum_{k}a_{i,k}b_{j,k}^T$

On ajoute une opération de transposition à notre structure ```Matrix```
```rust
impl<T: Copy> Matrix<T> {
    fn transpose(&mut self) {
        let mut transpose = Matrix::new_uninit(self.height, self.width);
        for i in 0..self.height {
            for j in 0..self.width {
                transpose[(j, i)] = self[(i, j)];
            }
        }
        self.data = transpose.data;
        let height = self.height;
        self.height = self.width;
        self.width = height;
    }
}
```

on l'utilise comme ceci
```rust
pub fn transpose_multiply<T>(a: &Matrix<T>, b: &mut Matrix<T>) -> Matrix<T>
where
    T: Sum + AddAssign<T> + Mul<T, Output = T> + Copy + Default,
{
    assert_eq!(a.width, b.height);
    let mut c = Matrix::new_uninit(b.width, a.height);
    b.transpose();
 
    for i in 0..c.height {
        for j in 0..c.width {
            c[(i, j)] = (0..a.width).into_iter().map(|k| a[(i,k)]*b[(j,k)]).sum();
        }
    }
    c
}
```

On teste avec le ```main``` suivant
```rust
fn main() {

    let half_log_grid_size = 10;
    let log_grid_size = 2*half_log_grid_size;
    let size = 1 << half_log_grid_size;
 
    // with transpose
    let a = Matrix::new(size, size, vec![1; size * size]);
    let mut b = Matrix::new(size, size, vec![1; size * size]);
    assert_eq!(size, transpose_multiply(&a, &mut b).width);
 
}
```

Pour ```time``` on obtient un résultat de **0m0,780s** et pour ```cachegrind``` **D1  miss rate:            3.2% (          3.1%     +      46.8%  )**. On voit donc qu'avec la capacité de transposer, on peut considérablement améliorer les performances malgré le surcout de calcul pour faire la transposition même.

On peut cependant encore mieux utiliser le cache.
Pour l'instant on parcourt $c_{0,0}$ puis $c_{0,1}$ etc.
Si on recapitule les lignes mises à contribution pour les 4 premières valeurs de $c_{i,j}$ on a:
* $c_{0,0}$ a besoin de $a_{0,*}$ et $b_{0,*}$
* $c_{0,1}$ a besoin de $a_{0,*}$ et $b_{1,*}$
* $c_{0,2}$ a besoin de $a_{0,*}$ et $b_{2,*}$
* $c_{0,3}$ a besoin de $a_{0,*}$ et $b_{3,*}$

On a ainsi requis 5 lignes différentes : $a_{0,*};b_{0,*};b_{1,*};b_{2,*};b_{3,*}$

Si on imaginait un autre parcours pour les $c_{i,j}$ qui donnerait par exemple comme 4 premières valeurs : $c_{0,0},c_{0,1},c_{1,1},c_{1,0}$ on aurait alors:
* $c_{0,0}$ a besoin de $a_{0,*}$ et $b_{0,*}$
* $c_{0,1}$ a besoin de $a_{0,*}$ et $b_{1,*}$
* $c_{1,1}$ a besoin de $a_{1,*}$ et $b_{1,*}$
* $c_{1,0}$ a besoin de $a_{1,*}$ et $b_{0,*}$

On aurait ainsi non plus 5 mais 4 lignes différentes requises : $a_{0,*};a_{1,*}; b_{0,*};b_{1,*}$ ce qui améliore notre utilisation du cache !

## Hilbert

On va dans cette optique utiliser la courbe de Hilbert afin de parcourir l'espace plus efficacement du point de vue du cache.
Pour calculer l'inverse de la transformée de Hilbert on peut utiliser l'automate suivant : ![automata](https://github.com/TC5027/blog/blob/master/pngs/automata.png)
Je représente l'automate comme un enum avec comme variant les états, implémentant une fonction de transition ```next``` :

```rust
enum InvHilbertAutomata {
    State0,
    State1,
    State2,
    State3,
}
 
impl InvHilbertAutomata {
    pub fn new() -> Self {
        InvHilbertAutomata::State3
    }
 
    pub fn next(&mut self, pair: u64, i: &mut usize, j: &mut usize) {
        *self = match &self {
            InvHilbertAutomata::State0 => match pair {
                0 => {*i = *i << 1 | 1;*j = *j << 1 | 1;InvHilbertAutomata::State1}
                1 => {*i = *i << 1;*j = *j << 1 | 1;InvHilbertAutomata::State0}
                2 => {*i = *i << 1;*j = *j << 1;InvHilbertAutomata::State0}
                3 => {*i = *i << 1 | 1;*j = *j << 1;InvHilbertAutomata::State3}
                _ => {panic!()}
            },
            InvHilbertAutomata::State1 => match pair {
                0 => {*i = *i << 1 | 1;*j = *j << 1 | 1;InvHilbertAutomata::State0}
                1 => {*i = *i << 1 | 1;*j = *j << 1;InvHilbertAutomata::State1}
                2 => {*i = *i << 1;*j = *j << 1;InvHilbertAutomata::State1}
                3 => {*i = *i << 1;*j = *j << 1 | 1;InvHilbertAutomata::State2}
                _ => {panic!()}
            },
            InvHilbertAutomata::State2 => match pair {
                0 => {*i = *i << 1;*j = *j << 1;InvHilbertAutomata::State3}
                1 => {*i = *i << 1 | 1;*j = *j << 1;InvHilbertAutomata::State2}
                2 => {*i = *i << 1 | 1;*j = *j << 1 | 1;InvHilbertAutomata::State2}
                3 => {*i = *i << 1;*j = *j << 1 | 1;InvHilbertAutomata::State1}
                _ => {panic!()}
            },
            InvHilbertAutomata::State3 => match pair {
                0 => {*i = *i << 1;*j = *j << 1;InvHilbertAutomata::State2}
                1 => {*i = *i << 1;*j = *j << 1 | 1;InvHilbertAutomata::State3}
                2 => {*i = *i << 1 | 1;*j = *j << 1 | 1;InvHilbertAutomata::State3}
                3 => {*i = *i << 1 | 1;*j = *j << 1;InvHilbertAutomata::State0}
                _ => {panic!()}
            },
        };
    }
}
```

On peut maintenant définir une fonction pour la transformée inverse
```rust
fn inv_hilbert(h: u64, mut log_grid_size: u8) -> (usize, usize) {
    let (mut i, mut j) = (0, 0);
 
    let mut automata = InvHilbertAutomata::new();
    let mut mask = 1 << (log_grid_size - 1) | 1 << (log_grid_size - 2);
    while mask > 0 {
        automata.next((h & mask) >> (log_grid_size - 2), &mut i, &mut j);
        log_grid_size -= 2;
        mask >>= 2;
    }
    (i, j)
}
```

Pour illustrer le bon fonctionnement
```rust
fn main() {
    let half_log_grid_size = 4;
    let log_grid_size = 2*half_log_grid_size;
    let size = 1 << half_log_grid_size;
    assert_eq!(16,size);
    let mut parcours: Vec<Vec<usize>> = (0..16).into_iter().map(|_| vec![0;size]).collect();
    for k in 0..size*size {
        let (i,j) = inv_hilbert(k as u64, log_grid_size);
        parcours[i][j] = k;
    }
    for line in parcours {
        for order in line {
            print!("{:0>3} ",order);
        }
        println!();
    }
}
```

On définit une nouvelle fonction de multiplication, parcourant les données selon cette courbe
```rust
pub fn hilbert_multiply<T>(a: &Matrix<T>, b: &mut Matrix<T>, log_grid_size: u8) -> Matrix<T>
where
    T: Sum + AddAssign<T> + Mul<T, Output = T> + Copy + Default,
{
    assert_eq!(a.width, b.height);
    let mut c = Matrix::new_uninit(b.width, a.height);
    assert_eq!(1<<(log_grid_size/2),c.width);
    assert_eq!(1<<(log_grid_size/2),c.height);
    b.transpose();

    for h in 0..c.height*c.width {
        let (i,j) = inv_hilbert(h as u64, log_grid_size);
        c[(i,j)] = (0..a.width).into_iter().map(|k| a[(i,k)]*b[(j,k)]).sum();
    }
    c
}
```

On teste comme suit
```rust
fn main() {
    let half_log_grid_size = 10;
    let log_grid_size = 2*half_log_grid_size;
    let size = 1 << half_log_grid_size;
 
    // with hilbert
    let a = Matrix::new(size, size, vec![1; size * size]);
    let mut b = Matrix::new(size, size, vec![1; size * size]);
    assert_eq!(size, hilbert_multiply(&a, &mut b, log_grid_size).width);
 
}
```

On obtient comme résultat avec ```time``` **0m0,816s** et avec ```cachegrind``` **D1  miss rate:            0.8% (          0.8%     +      23.9%  )**. On a donc amélioré notre utilisation du cache mais le temps d'éxécution a augmenté, de par le calcul supplémentaire pour la transformée inverse.
On peut voir avec ce flamegraph ![flame](https://github.com/TC5027/blog/blob/master/pngs/search_inv.png) la part qu'il représente environ 4% des calls observés

Si on passe à ```half_log_grid_size = 11``` on a cette fois pour transpose **0m6,178s** et pour hilbert **0m6,219s**

## SIMD

Une autre amélioration possible consiste à utiliser [SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data). En effet, cela se prete bien à notre cas car on fait un grand nombre de multiplications, qu'on somme toutes ensuite.

Je vais utiliser pour cela le module ```std::simd``` qui a l'heure où j'écris, est uniquement disponible sur ```nightly```.

Pour ça, on peut faire ```rustup install nightly```, avec ```rustup show``` on doit voir ```nightly``` dans installed toolchains, on passe ensuite ```nightly``` en default avec ```rustup default nightly``` (```rustup default stable``` pour revenir en arrière) et on est pret.

A l'heure où j'écris, ```rustup show``` me montre ```rustc 1.62.0-nightly```

Pour utiliser ```std::simd``` on doit également mettre dans ```main.rs``` ```#![feature(portable_simd)]```

On commence par ajouter une nouvelle implémentation pour ```Index``` afin de pouvoir facilement accéder à des lignes de nos matrices.
```rust
impl<T> Index<usize> for Matrix<T> {
    type Output = [T];
    fn index(&self, index: usize) -> &[T] {
        &self.data[self.width*index..self.width*(index+1)]
    }
}
```

La multiplication peut s'écrire ainsi
```rust
use  std::simd::u32x16;
pub fn hilbert_multiply_simd(a: &Matrix<u32>, b: &mut Matrix<u32>, log_grid_size: u8) -> Matrix<u32>
{
    assert_eq!(a.width, b.height);
    let mut c = Matrix::new_uninit(b.width, a.height);
    assert_eq!(1<<(log_grid_size/2),c.width);
    assert_eq!(1<<(log_grid_size/2),c.height);
    assert!(a.width%16==0);
    b.transpose();

    for h in 0..c.height*c.width {
        let (i,j) = inv_hilbert(h as u64, log_grid_size);
        let a_row = &a[i];
        let b_row = &b[j];
        let mut sum = u32x16::splat(0);
        for k in (0..a.width).step_by(16) {
            sum += u32x16::from_slice(&a_row[k..])*u32x16::from_slice(&b_row[k..]);
        }
        c[(i,j)] = sum.reduce_sum();
    }
    c
}
```

Avec le ```main``` suivant
```rust
fn main() {
    
    let half_log_grid_size = 11;
    let log_grid_size = 2*half_log_grid_size;
    let size = 1 << half_log_grid_size;

	// with simd
    let a = Matrix::new(size, size, vec![1u32; size * size]);
    let mut b = Matrix::new(size, size, vec![1u32; size * size]);
    assert_eq!(size, hilbert_multiply_simd(&a, &mut b, log_grid_size).width);
}
```

on a comme résultat avec ```time``` **0m2,434s** !

## Lindenmayer

Pour poursuivre notre optimisation, on peut remarquer que deux entiers successifs partagent un préfixe commun pour ce qui est de leurs représentations binaires. Ainsi avec notre fonction pour calculer la transformée inverse, on a des transitions similaires ce qui est donc une perte de temps, on répète les mêmes transitions entre deux appels successifs, et pourrait donc etre optimiser.

Le papier [suivant](https://eprints.cs.univie.ac.at/5726/1/loops.pdf) présente un système de Lindenmayer pour palier à ce problème. Le code pour ça se trouve [ici](https://github.com/TC5027/matmul_singlethread) et on a comme résultat avec ```time``` pour ```half_log_grid_size = 11``` **0m1,894s**. :)

