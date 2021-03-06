# TARGET_LATENCY_2

The work stealing policy  is based on local deques (one deque for each thread). To implement this we can use the macro ```thread_local!```
```rust
thread_local!{
    static local_deque: RefCell<VecDeque<Box<dyn Fn()+Send+'static>>> =
        RefCell::new(VecDeque::new());
}
```

We can now change the code of ```forall``` to split our requests into tasks. The requests we work with consist of repeating X times a task and so a request processed by a thread must provide this "parallelism" at the level of thread's local deque by filling it with X tasks instead of executing all of it directly like it is the case currently. We change the code of ```forall``` :
```rust
pub fn forall<T: Fn() + Send + Clone + 'static>(&mut self, repetitions: usize, task: T) {
    self.global_queue
        .lock()
        .unwrap()
        .push_back(Box::new(move || {
            local_deque.with(|deque| {
                for _ in 0..repetitions {
                    let task = task.clone();
                    deque.borrow_mut().push_back(Box::new(move || task()));
                }
            })
        }));
}
```

For the tasks, given the repetition , we added the ```Clone``` trait on the parameter ```T``` (makes sense given our framework).

Threads must now feed themselves from global queue **and** from their local deque. We need to change ```feed_and_execute``` in order to incorporate the mechanism we described in introduction with *steal-first*.
Our threads must first check if their local deque is empty or not. If it contains a task, we pick it and we perform it. Otherwise, if there exist tasks in the system, in one of the local deque of the other threads, we try to find it and steal if to process it. If there is no task in the system we pick a request from the global queue.

Local deques must be accessible from the other threads, we will need to change its type, encompassing the previous one contained in ```RefCell``` into ```Arc<Mutex<..>>```. We create an alias for this type
```rust
type LocalDeque = Arc<Mutex<VecDeque<Box<dyn Fn()+Send+'static>>>>
```

```feed_and_execute``` will so take a new argument which corresponds to a ```Vec``` of local deques present in the system. We change also the constructor of ```Threadpool``` by initializing the local deques before thread's creation. We so have a ```Vec<Arc<Mutex<Deque<..>>>>``` that we clone for each thread and pass as argument to ```feed_and_execute``` inside the thread.
```rust
pub fn new(number_of_threads: usize) -> Self {
    let global_queue = Arc::new(Mutex::new(VecDeque::new()));
    let local_deques: Vec<LocalDeque> = (0..number_of_threads).map(|_| {
        Arc::new(Mutex::new(VecDeque::new()))
    }).collect();
    let handlers = (0..number_of_threads)
        .map(|i| {
            let global_queue = global_queue.clone();
            let local_deques = local_deques.clone();
            spawn(move || {
                local_deque.with(|deque| {
                    *deque.borrow_mut() = local_deques[i].clone();
                });
                feed_and_execute(global_queue, local_deques, i)
            })
        })
        .collect();
    Threadpool {
        handlers,
        global_queue,
    }
}
```

The logic inside ```feed_and_execute``` evolves also in order to reflect the behavior we described above. We need to keep track of the tasks' presence inside the system, which we do thanks to a global counter ```tasks_counter: Arc<AtomicUsize>``` initialized at 0.
```rust
struct Threadpool {
    handlers: Vec<JoinHandle<()>>,
    global_queue: Shared<VecDeque<Box<dyn Fn() + Send + 'static>>>,
    tasks_counter: Arc<AtomicUsize>,
}
```

This counter is then handled inside ```forall```. At request's initialization, we increment of ```repetitions``` and for each task completed, we decrement by 1.
```rust
pub fn forall<T: Fn() + Send + Clone + 'static>(&mut self, repetitions: usize, task: T) {
    let tasks_counter = self.tasks_counter.clone();
    self.global_queue
        .lock()
        .unwrap()
        .push_back(Box::new(move || {
            local_deque.with(|deque| {
                for _ in 0..repetitions {
                    let task = task.clone();
                    let tasks_counter = tasks_counter.clone();
                    deque.borrow_mut().lock().unwrap().push_back(Box::new(move || {
                        task();
                        tasks_counter.fetch_sub(1,Relaxed);
                    }));
                }
            });
            tasks_counter.fetch_add(repetitions,Relaxed);
        }));
}
```

Finally, we have everything we need to implement the logic inside ```feed_and_execute```
```rust
fn feed_and_execute(global_queue: Shared<VecDeque<Box<dyn Fn() + Send + 'static>>>, local_deques: Vec<LocalDeque>, tasks_counter: Arc<AtomicUsize>, local_index: usize) {
    let mut rng = thread_rng();
    loop {
        // if the local_deque is empty then we accept a new request from
        // the global queue OR we steal a task from another deque
        let local_is_empty = local_deques[local_index].lock().unwrap().is_empty();
        if local_is_empty {
            // there are tasks in the system that we can steal
            if tasks_counter.load(Relaxed)!=0 {
                // we pick a random target
                let mut index;
                loop {
                    index = rng.gen::<usize>()%local_deques.len();
                    if index!=local_index {break;}
                }
                let option_task = local_deques[index].lock().unwrap().pop_back();
                if option_task.is_some() {
                    option_task.unwrap()();
                }
            }
            // no more tasks in the system that we can steal so we pick a new request from
            // the global queue
            else {
                // INITIALIZATION : we get the request which only spreads its
                // inner tasks into the deque local to the thread
                let option_request = global_queue.lock().unwrap().pop_front();
                if option_request.is_some() {
                    option_request.unwrap()();
                }
            }
        } else {
            // we get a task from the local_deque and perform it
            let option_task = local_deques[local_index].lock().unwrap().pop_back();
            if option_task.is_some() {
                option_task.unwrap()();
            }
        }
    }
}
```

At this very moment, our whole code looks like this (rand = "0.8.0")
```rust
use std::boxed::Box;
use std::cell::RefCell;
use std::collections::VecDeque;
use std::sync::{Arc, Mutex, atomic::{Ordering::Relaxed,AtomicUsize}};
use std::thread::{sleep, spawn, JoinHandle};
use rand::{thread_rng, Rng};
use std::time::{Instant,Duration};

type Shared<T> = Arc<Mutex<T>>;
type LocalDeque = Arc<Mutex<VecDeque<Box<dyn Fn()+Send+'static>>>>;

thread_local! {
    static local_deque: RefCell<LocalDeque> =
        RefCell::new(Arc::new(Mutex::new(VecDeque::new())));
}

fn feed_and_execute(global_queue: Shared<VecDeque<Box<dyn Fn() + Send + 'static>>>, local_deques: Vec<LocalDeque>, tasks_counter: Arc<AtomicUsize>, local_index: usize) {
    let mut rng = thread_rng();
    loop {
        // if the local_deque is empty then we accept a new request from
        // the global queue OR we steal a task from another deque
        let local_is_empty = local_deques[local_index].lock().unwrap().is_empty();
        if local_is_empty {
            // there are tasks in the system that we can steal
            if tasks_counter.load(Relaxed)!=0 {
                // we pick a random target
                let mut index;
                loop {
                    index = rng.gen::<usize>()%local_deques.len();
                    if index!=local_index {break;}
                }
                let option_task = local_deques[index].lock().unwrap().pop_back();
                if option_task.is_some() {
                    option_task.unwrap()();
                }
            }
            // no more tasks in the system that we can steal so we pick a new request from
            // the global queue
            else {
                // INITIALIZATION : we get the request which only spreads its
                // inner tasks into the deque local to the thread
                let option_request = global_queue.lock().unwrap().pop_front();
                if option_request.is_some() {
                    option_request.unwrap()();
                }
            }
        } else {
            // we get a task from the local_deque and perform it
            let option_task = local_deques[local_index].lock().unwrap().pop_back();
            if option_task.is_some() {
                option_task.unwrap()();
            }
        }
    }
}

struct Threadpool {
    handlers: Vec<JoinHandle<()>>,
    global_queue: Shared<VecDeque<Box<dyn Fn() + Send + 'static>>>,
    tasks_counter: Arc<AtomicUsize>,
}

impl Threadpool {
    pub fn new(number_of_threads: usize) -> Self {
        let global_queue = Arc::new(Mutex::new(VecDeque::new()));
        let tasks_counter = Arc::new(AtomicUsize::new(0));
        let local_deques: Vec<LocalDeque> = (0..number_of_threads).map(|_| {
            Arc::new(Mutex::new(VecDeque::new()))
        }).collect();
        let handlers = (0..number_of_threads)
            .map(|i| {
                let global_queue = global_queue.clone();
                let local_deques = local_deques.clone();
                let tasks_counter = tasks_counter.clone();
                spawn(move || {
                    local_deque.with(|deque| {
                        *deque.borrow_mut() = local_deques[i].clone();
                    });
                    feed_and_execute(global_queue, local_deques, tasks_counter, i)
                })
            })
            .collect();
        Threadpool {
            handlers,
            global_queue,
            tasks_counter,
        }
    }

    pub fn forall<T: Fn() + Send + Clone + 'static>(&mut self, repetitions: usize, task: T) {
        let tasks_counter = self.tasks_counter.clone();
        let enter_time = Instant::now();
        self.global_queue
            .lock()
            .unwrap()
            .push_back(Box::new(move || {
                local_deque.with(|deque| {
                    for _ in 0..repetitions {
                        let task = task.clone();
                        let tasks_counter = tasks_counter.clone();
                        deque.borrow_mut().lock().unwrap().push_back(Box::new(move || {
                            task();
                            tasks_counter.fetch_sub(1,Relaxed);
                        }));
                    }
                });
                tasks_counter.fetch_add(repetitions,Relaxed);
            }));
    }
}
```

We can make a demonstration to show that we have the expected behavior with the following ```main``` :
```rust
fn  main() {
    let  mut  threadpool = Threadpool::new(4);
    threadpool.forall(8, || sleep(std::time:uration::from_secs(2)));
    sleep(std::time:uration::from_secs(5));
}
```

having added ```println!``` in ```feed_and_execute``` where we perform the task
```rust
println!("{:?} - stolen/local task performed", std::time::Instant::now());
option_task.unwrap()();
```

we get the following output :
```console
PS C:\X\truct> cargo run
   Compiling truct v0.1.0 (C:\X\truct)
    Finished dev [unoptimized + debuginfo] target(s) in 0.53s
     Running `target\debug\truct.exe`
Instant { t: 3030.4576691s } - local task performed
Instant { t: 3030.4578215s } - stolen task performed
Instant { t: 3030.4578237s } - stolen task performed
Instant { t: 3030.4578546s } - stolen task performed
Instant { t: 3032.4758277s } - local task performed
Instant { t: 3032.4758412s } - stolen task performed
Instant { t: 3032.4758552s } - stolen task performed
Instant { t: 3032.4758626s } - stolen task performed
```

Work-stealing is highlighted, the tasks got distributed among the threads. With 4 threads inside the threadpool, we execute the whole request, which is decoupled into 8 tasks, in 2 "steps" if we look at the timings.

We can also illustrate the harmful phenomenon that we described in introduction if we add before the ```for``` loop in ```forall``` :
```rust
deque.borrow_mut().lock().unwrap().push_back(Box::new(move || {
    println!("it took {:?} to execute the request since entering the system", enter_time.elapsed());
}));
```

Our previous code now produce the following output :
```console
PS C:\X\truct> cargo run
   Compiling truct v0.1.0 (C:\X\truct)
    Finished dev [unoptimized + debuginfo] target(s) in 0.53s
     Running `target\debug\truct.exe`
Instant { t: 9029.6603757s } - local task performed
Instant { t: 9029.6604427s } - stolen task performed
Instant { t: 9029.6604687s } - stolen task performed
Instant { t: 9029.6605509s } - stolen task performed
Instant { t: 9031.663556s } - local task performed
Instant { t: 9031.6635803s } - stolen task performed
Instant { t: 9031.6783021s } - stolen task performed
Instant { t: 9031.6783185s } - stolen task performed
Instant { t: 9033.6695636s } - stolen task performed
it took 4.0102887s to execute the request since entering the system
```

If we consider this ```main```, saying that the target latency is of 5 seconds
```rust
fn  main() {
    let  mut  threadpool = Threadpool::new(4);
    threadpool.forall(20, || sleep(std::time:uration::from_secs(2)));
    sleep(std::time:uration::from_secs(1));
    threadpool.forall(4, || sleep(std::time:uration::from_secs(2)));
    sleep(std::time:uration::from_secs(60));
}
```

We have this output :

```console
PS C:\X\truct> cargo run
   Compiling truct v0.1.0 (C:\X\truct)
    Finished dev [unoptimized + debuginfo] target(s) in 0.53s
     Running `target\debug\truct.exe`
Instant { t: 13057.9065697s } - local task performed
Instant { t: 13057.9066483s } - stolen task performed
Instant { t: 13057.9066518s } - stolen task performed
Instant { t: 13057.9066949s } - stolen task performed
Instant { t: 13059.9172146s } - local task performed
Instant { t: 13059.9172644s } - stolen task performed
Instant { t: 13059.917269s } - stolen task performed
Instant { t: 13059.9172727s } - stolen task performed
Instant { t: 13061.9296904s } - local task performed
Instant { t: 13061.9297347s } - stolen task performed
Instant { t: 13061.9297352s } - stolen task performed
Instant { t: 13061.9297385s } - stolen task performed
Instant { t: 13063.9419235s } - stolen task performed
Instant { t: 13063.9419454s } - local task performed
Instant { t: 13063.9419463s } - stolen task performed
Instant { t: 13063.941949s } - stolen task performed
Instant { t: 13065.952939s } - local task performed
Instant { t: 13065.9529403s } - stolen task performed
Instant { t: 13065.9529413s } - stolen task performed
Instant { t: 13065.9529761s } - stolen task performed
Instant { t: 13067.9616301s } - stolen task performed
it took 10.0563955s to execute the request since entering the system
Instant { t: 13067.9636124s } - stolen task performed
Instant { t: 13067.9616624s } - stolen task performed
Instant { t: 13067.9616596s } - local task performed
Instant { t: 13067.9616602s } - stolen task performed
Instant { t: 13069.9713302s } - stolen task performed
it took 11.0513439s to execute the request since entering the system
```

The first request (which would have missed the target latency anyway) has monopolized all the threads for its full execution and the second request, which could have hit target latency, misses it also.


