# Hilbert curve for fast matrix multiplication (single thread)

## Matrix multiplication

In this article we will focus on matrix multiplication for matrices of size a power of 2, and how to improve speed of execution.

We begin by setting a struc for matrices
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

We also implement ```Index``` and ```IndexMut``` traits to access data
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

We can now write a function for the multiplication, starting from the classic mathematical definition
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

To test we set the following ```main```
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

With the ```time``` command (after having built in release mode) we get a result of **0m4,375s**
One of the key point for performance is an optimal usage of cache, meaning go accross the data in an order that avoid having many cache misses. We can measure it with ```valgrind --tool=cachegrind```, which outputs for our program
**D1  miss rate:           50.0% (         50.0%     +      73.1%  )**. The first value inside the parenthesis represents the percentage of readings accessing data not in cache, the second value goes for writings. The value before the parenthesis represents the percentage of accesses (readings+writings) to data not in cache.
We see from this result that it is not optimal, indeed we multiply lines with columns. Data inside a column are not contiguous in memory given the line-representation of ```Matrix``` which explains the bad behavior for cache.

## Transpose

To better navigate the data, we can first transpose the second matrix to only work on lines !

<img src="https://render.githubusercontent.com/render/math?math=c_{i,j} = \sum_{k}a_{i,k}b_{k,j} = \sum_{k}a_{i,k}b_{j,k}^T">

We add a transposition operation to our struct ```Matrix```
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

We use it like this
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

We test with the following ```main```
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

With ```time``` we get a result of **0m0,780s** and with ```cachegrind``` **D1  miss rate:            3.2% (          3.1%     +      46.8%  )**. We see that with the capacity to transpose, we can drastically improve the performances even though we add work to perform the transposition itself.

Still we can improve cache usage.
Right now we navigate first c_{0,0} then c_{0,1} etc.
If we sum up the lines we use for the first 4 values of c_{i,j} we have:
* c_{0,0} needs a_{0,} and b_{0,}
* c_{0,1} needs a_{0,} and b_{1,}
* c_{0,2} needs a_{0,} and b_{2,}
* c_{0,3} needs a_{0,} and b_{3,}

We require 5 different lines : a_{0,};b_{0,};b_{1,};b_{2,};b_{3,}

If we imagine another navigation for the $c_{i,j}$ which would produce as the 4 first values : c_{0,0},c_{0,1},c_{1,1},c_{1,0} we would have:
* c_{0,0} needs a_{0,} and b_{0,}
* c_{0,1} needs a_{0,} and b_{1,}
* c_{1,1} needs a_{1,} and b_{1,}
* c_{1,0} needs a_{1,} and b_{0,}

We would have not 5 but 4 different lines required in that case : a_{0,};a_{1,};b_{0,};b_{1,} which is better regarding the cache !

## Hilbert

We will use Hilbert curbe in order to navigate space more efficiently for cache. To compute the invert of the Hilbert transform we can use the following automata : 

![automata](https://github.com/TC5027/blog/blob/master/pngs/automata.png)

I represented the automata like an enum with the states being its variants, implementing a function of transition ```next``` :

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

We can now write a function for the inverse transform
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

To 'prove' the correctness :
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

We define a new function for multiplication, navigating data according to the curve
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

We test with this
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

We get as result with ```time``` **0m0,816s** and with ```cachegrind``` **D1  miss rate:            0.8% (          0.8%     +      23.9%  )**. We improved our usage of cache but time of execution increased, due to the additional work of computing the inverse transform.
We can see with this flamegraph ![flame](https://github.com/TC5027/blog/blob/master/pngs/search_inv.png) that it represents 4% of the traces.

If we set ```half_log_grid_size = 11``` we get this time for transpose **0m6,178s** and for hilbert **0m6,219s**

## SIMD

Another improvement could be using [SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data). Actually, it fits well to our case as we perform many multiplications followed by a sum.

To do so I will use the module ```std::simd``` which is only available on ```nightly``` while I'm writing.

We do ```rustup install nightly```, with ```rustup show``` we should see ```nightly``` in installed toolchains, then we set ```nightly``` as default with ```rustup default nightly``` (```rustup default stable``` to go back) and we are ready.

As I write, ```rustup show``` shows ```rustc 1.62.0-nightly```

To use ```std::simd``` we must also write in ```main.rs``` ```#![feature(portable_simd)]```

We start by adding another implementation for ```Index``` in order to easily access data from lines of our matrices.
```rust
impl<T> Index<usize> for Matrix<T> {
    type Output = [T];
    fn index(&self, index: usize) -> &[T] {
        &self.data[self.width*index..self.width*(index+1)]
    }
}
```

Multiplication can be written like so
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

With the following ```main```
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

we get as result with ```time``` **0m2,434s** !

## Lindenmayer

To go on with our optimization, we can notice that two consecutive integers share a common prefix regarding their binary representation. Because of it, our function to compute the inverse transform performs similar initial transition for the two, which is a waste of time.

This [paper](https://eprints.cs.univie.ac.at/5726/1/loops.pdf) presents a Lindenmayer system to answer our need. My code for this is [here](https://github.com/TC5027/matmul_singlethread) and we have a result of **0m1,894s** for ```half_log_grid_size = 11``` :)
