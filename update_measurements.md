# An example on measuring part of your code with callgrind

## Scenario

Consider this structure, corresponding to a grocery product for example.

```rust
struct Product {
    name: String,
    price: f64,
    vendor: String,
    production_site: String,
}
```

Suppose now that we want to update all the products of our grocery. We define a function ```update_1``` performing this task.

```rust
impl Product {
    fn update_product(&mut self) {
        process_name(&mut self.name);
        process_price(&mut self.price);
        process_vendor(&mut self.vendor);
        process_production_site(&mut self.production_site);
    }
}

fn update_1(mut products: Vec<Product>) {
    for product in products.iter_mut() {
        product.update_product();
    }
}
```

Each ```update_product``` call being concurrent, we could imagine sharing the work between 2 threads which is done in a new function ```update_2```, used on a result of ```split_at_mut``` applied on our ```Vec<Product>```.

```rust
use crossbeam_utils::thread;

fn update_2(subpart1: &mut [Product], subpart2: &mut [Product]) {
    thread::scope(|scope| {
        let handler1 = scope.spawn(move |_| {
            for product in subpart1 {
                product.update_product();
            }
        });
        let handler2 = scope.spawn(move |_| {
            for product in subpart2 {
                product.update_product();
            }
        });
        handler1.join().unwrap();
        handler2.join().unwrap();
    })
    .unwrap();
}
```

Finally, the last way of updating I could think of is splitting our structure in 2 and having a thread update name and price and another thread updating vendor and production_site. This is the idea behing ```update_3```

```rust
struct SubProduct1 {
    name: String,
    price: f64,
}

impl SubProduct1 {
    fn update_subproduct(&mut self) {
        process_name(&mut self.name);
        process_price(&mut self.price);
    }
}

struct SubProduct2 {
    vendor: String,
    production_site: String,
}

impl SubProduct2 {
    fn update_subproduct(&mut self) {
        process_vendor(&mut self.vendor);
        process_production_site(&mut self.production_site);
    }
}

fn update_3(mut sproduct1s: Vec<SubProduct1>, mut sproduct2s: Vec<SubProduct2>) {
    thread::scope(|scope| {
        let handler1 = scope.spawn(move |_| {
            for sproduct1 in sproduct1s.iter_mut() {
                sproduct1.update_subproduct();
            }
        });
        let handler2 = scope.spawn(move |_| {
            for sproduct2 in sproduct2s.iter_mut() {
                sproduct2.update_subproduct();
            }
        });
        handler1.join().unwrap();
        handler2.join().unwrap();
    })
    .unwrap();
}
```

What would the best choice in terms of performance and why ? Of course it depends on the functions ```process_name```, ```process_price```, ```process_vendor``` and ```process_production_site```. We will work with this.

```rust
fn process_name(name: &mut String) {
    if name == "Hamburger" {
        name.clear();
    }
}

fn process_price(price: &mut f64) {
    if *price > 50.0 {
        *price *= 0.8;
    }
}

fn process_vendor(vendor: &mut String) {
    if vendor != "Heinz" {
        vendor.push_str("Nevermind");
    }
}

fn process_production_site(production_site: &mut String) {
    if production_site == "Georgia" {
        *production_site = String::from("Montpellier")
    } else if production_site == "Afghanistan" {
        *production_site = String::from("Henry Cavill")
    } else {
        *production_site = String::from("World")
    }
}
```

We will work with this set of products.

```rust
let names: Vec<&str> = vec!["Ketchup", "Mustard", "Rice", "Hamburger", "Cream"];
let prices: Vec<f64> = vec![1.45, 2.99, 6.9, 209.0, 55.2, 1.00, 0.0085];
let vendors: Vec<&str> = vec!["Uncle Ben", "Heinz", "McDonald"];
let production_sites: Vec<&str> = vec!["China", "Italy", "Afghanistan", "Georgia"];

let mut products: Vec<Product> = (0..10_000_000)
    .into_iter()
    .map(|i| Product {
        name: String::from(names[i % names.len()]),
        price: prices[i % prices.len()],
        vendor: String::from(vendors[i % vendors.len()]),
        production_site: String::from(production_sites[i % production_sites.len()]),
    })
    .collect();
```

## Measurements

Associated repository with the code : https://github.com/TC5027/rust_stuff/tree/master/updates

The results we get surrounding the calls to ```update_X``` with ```std::time::Instant::now()``` and ```.elasped()```

update_1 -> 475.314129ms 474.022887ms 487.448066ms 476.663291ms 521.97216ms
update_2 -> 297.573154ms 296.91471ms 307.319173ms 291.371916ms 315.54673ms
update_3 -> 1.362059389s 965.599649ms 966.835205ms 1.076025605s 977.364906ms

There are huge differences in the results but do we measure the same thing everytime ?

To inspect that we can use ```cargo flamegraph```, here is what we get

![FirstVersion](https://github.com/TC5027/blog/blob/master/pngs/flamegraph(update_1).svg)](https://github.com/TC5027/blog/blob/master/pngs/flamegraph(update_1).svg)

![SecondVersion](https://github.com/TC5027/blog/blob/master/pngs/flamegraph(update_2).svg)](https://github.com/TC5027/blog/blob/master/pngs/flamegraph(update_2).svg)

![ThirdVersion](https://github.com/TC5027/blog/blob/master/pngs/flamegraph(update_3).svg)](https://github.com/TC5027/blog/blob/master/pngs/flamegraph(update_3).svg)


Zooming on the ```update_X``` function in the flamegraphs we see that there is a huge part for 1 and 3 due to dropping the ```Vec```s. I changed the signatures of ```update_1``` and ```update_3``` to take as input references to avoid the drops inside these functions.

The results we get now for time of execution.

update_1 -> 235.201796ms 239.38725ms 234.74462ms 235.550715ms 241.097017ms
update_3 -> 288.582635ms 302.115826ms 287.761127ms 289.621371ms 292.264621ms

How can we explain these results ? We can use callgrind with simulate-cache to get an idea on how our program interact with cache mechanism. Except that using callgrind directly on our whole program wouldn't be pertinent as before calling ```update_X``` there is a huge step for preparing the data which differs according to the version.

Looking for answers I found out that we could insert assembly into our code corresponding to the start and end for callgrind. (https://stackoverflow.com/questions/32905212/how-to-use-kcachegrind-and-callgrind-to-measure-only-parts-of-my-code & https://github.com/lemonrock/callgrind)

I propose the following functions (x86_64)

```rust
use std::arch::asm;

const BASE: u64 = ((b'C' as u64) << 24) + ((b'T' as u64) << 16);

#[inline(always)]
pub unsafe fn do_client_request(default: u64, args: &[u64; 6]) -> u64 {
    let result;
    asm!(
    "rol rdi, 3
        rol rdi, 13
        rol rdi, 61
        rol rdi, 51
        xchg rbx,rbx",
    inout("rdx") default=>result,
    in("rax") args.as_ptr()
    );
    result
}

#[inline(always)]
pub fn start() {
    let (type_value, first_argument) = (BASE + 4, 0);
    unsafe { do_client_request(0, &[type_value, first_argument, 0, 0, 0, 0]) };
    let (type_value, first_argument) = (BASE + 2, 0);
    unsafe { do_client_request(0, &[type_value, first_argument, 0, 0, 0, 0]) };
}

#[inline(always)]
pub fn stop() {
    let (type_value, first_argument) = (BASE + 2, 0);
    unsafe { do_client_request(0, &[type_value, first_argument, 0, 0, 0, 0]) };
    let (type_value, first_argument) = (BASE + 5, 0);
    unsafe { do_client_request(0, &[type_value, first_argument, 0, 0, 0, 0]) };
}
```

For ```update_1``` we use it like this and launch ```valgrind --tool=callgrind --collect-atstart=no --instr-atstart=no --simulate-cache=yes ./target/release/updates```

```rust
fn update_1(products: &mut [Product]) {
    start();
    for product in products {
        product.update_product();
    }
    stop();
}
```

The results we get.
==8081== Events    : Ir Dr Dw I1mr D1mr D1mw ILmr DLmr DLmw
==8081== Collected : 3380143077 877190514 554857177 69 27501666 5 69 27501666 5
==8081== 
==8081== I   refs:      3,380,143,077
==8081== I1  misses:               69
==8081== LLi misses:               69
==8081== I1  miss rate:          0.00%
==8081== LLi miss rate:          0.00%
==8081== 
==8081== D   refs:      1,432,047,691  (877,190,514 rd + 554,857,177 wr)
==8081== D1  misses:       27,501,671  ( 27,501,666 rd +           5 wr)
==8081== LLd misses:       27,501,671  ( 27,501,666 rd +           5 wr)
==8081== D1  miss rate:           1.9% (        3.1%   +         0.0%  )
==8081== LLd miss rate:           1.9% (        3.1%   +         0.0%  )
==8081== 
==8081== LL refs:          27,501,740  ( 27,501,735 rd +           5 wr)
==8081== LL misses:        27,501,740  ( 27,501,735 rd +           5 wr)
==8081== LL miss rate:            0.6% (        0.6%   +         0.0%  )


For ```update_2``` writing a start/stop at the beginning/end of the function doesn't work, I guess because of the multi thread aspect. Reading callgrind documentation (https://valgrind.org/docs/manual/cl-manual.html) there is an option we could use : --separate-threads. What I have done :

```rust
fn update_2(subpart1: &mut [Product], subpart2: &mut [Product]) {
    start();
    thread::scope(|scope| {
        let handler1 = scope.spawn(move |_| {
            start();
            for product in subpart1 {
                product.update_product();
            }
            stop();
        });
        let handler2 = scope.spawn(move |_| {
            start();
            for product in subpart2 {
                product.update_product();
            }
            stop();
        });
        handler1.join().unwrap();
        handler2.join().unwrap();
    })
    .unwrap();
    stop();
}
```

And I launched ```valgrind --tool=callgrind --collect-atstart=no --instr-atstart=no --simulate-cache=yes --separate-threads=yes ./target/release/updates```.
I'm not totally sure of my methodology here so feel free to contact me if I'm wrong. The results we get.
==9066== Events    : Ir Dr Dw I1mr D1mr D1mw ILmr DLmr DLmw
==9066== Collected : 3519816954 918858923 608191726 424 27501869 87 423 27501869 87
==9066== 
==9066== I   refs:      3,519,816,954
==9066== I1  misses:              424
==9066== LLi misses:              423
==9066== I1  miss rate:          0.00%
==9066== LLi miss rate:          0.00%
==9066== 
==9066== D   refs:      1,527,050,649  (918,858,923 rd + 608,191,726 wr)
==9066== D1  misses:       27,501,956  ( 27,501,869 rd +          87 wr)
==9066== LLd misses:       27,501,956  ( 27,501,869 rd +          87 wr)
==9066== D1  miss rate:           1.8% (        3.0%   +         0.0%  )
==9066== LLd miss rate:           1.8% (        3.0%   +         0.0%  )
==9066== 
==9066== LL refs:          27,502,380  ( 27,502,293 rd +          87 wr)
==9066== LL misses:        27,502,379  ( 27,502,292 rd +          87 wr)
==9066== LL miss rate:            0.5% (        0.6%   +         0.0%  )

Following the same methodology for ```update_3``` we get.
==10101== Events    : Ir Dr Dw I1mr D1mr D1mw ILmr DLmr DLmw
==10101== Collected : 3553821924 910002896 608192648 447 29501476 133 447 29501476 133
==10101== 
==10101== I   refs:      3,553,821,924
==10101== I1  misses:              447
==10101== LLi misses:              447
==10101== I1  miss rate:          0.00%
==10101== LLi miss rate:          0.00%
==10101== 
==10101== D   refs:      1,518,195,544  (910,002,896 rd + 608,192,648 wr)
==10101== D1  misses:       29,501,609  ( 29,501,476 rd +         133 wr)
==10101== LLd misses:       29,501,609  ( 29,501,476 rd +         133 wr)
==10101== D1  miss rate:           1.9% (        3.2%   +         0.0%  )
==10101== LLd miss rate:           1.9% (        3.2%   +         0.0%  )
==10101== 
==10101== LL refs:          29,502,056  ( 29,501,923 rd +         133 wr)
==10101== LL misses:        29,502,056  ( 29,501,923 rd +         133 wr)
==10101== LL miss rate:            0.6% (        0.7%   +         0.0%  )

What we see is that there are less instructions executed for ```update_1```, probably due to the overhead of thread creation? Doing less work takes less time also. Otherwise we don't see much difference between ```update_2``` and ```update_3``` callgrind results.

