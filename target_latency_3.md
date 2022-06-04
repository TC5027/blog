# TARGET_LATENCY_3

In this last part, we will build a mechanism to prevent tasks from a request having a time of execution greater than the target latency, to be stolen. That way, we let other threads free to process new requests. To go deeper on this subject I recommend the reading of this [paper](https://www.cse.wustl.edu/~angelee/home_page/papers/steal.pdf).

We need to measure time of execution of a request dispatched into its inner tasks. To do so, we create a struct dedicated to tasks ```Task```
```rust
pub struct Task {
    inner_task : Box<dyn Fn() + Send + 'static>,
    request_declaration : Instant,
    counter_left_tasks : Arc<AtomicUsize>,
}
```

There is a field corresponding to the task to execute and a field corresponding to the moment the request responsible for this task has been declared. The thirds field corresponds to the number of tasks, originated from the father request, not yet executed.

We will need 3 methods for this structure :
* a constructor
* a method to execute the task
* a method to check if the task can be stolen or not

```rust
impl Task {
    pub fn new<T: Fn() + Send + 'static>(inner_task: T, request_declaration: Instant, counter_left_tasks: Arc<AtomicUsize>) -> Self {
        Self {
            inner_task: Box::new(inner_task),
            request_declaration,
            counter_left_tasks,
        }
    }

    pub fn execute(self) {
        todo!()
    }

    pub fn is_stealable(&self) -> bool {
        todo!()
    }
}
```

In order to include this *controlled work stealing*, we need to change the level where the tasks are executed. Actually, with the current code, what we get from local deques is directly the function to execute. What we want to do now is rather fill deques with ```Task``` instances which we could first analyze and then maybe execute.

The alias ```LocaDeque``` becomes ```type  LocalDeque = Arc<Mutex<VecDeque<Task>>>;``` and we change ```forall```.
```rust
pub fn forall<T: Fn() + Send + Clone + 'static>(&mut self, repetitions: usize, task: T) {
    let tasks_counter = self.tasks_counter.clone();
    let request_declaration = Instant::now();
    self.global_queue
        .lock()
        .unwrap()
        .push_back(Box::new(move || {
            local_deque.with(|deque| {
                let counter_left_tasks = Arc::new(AtomicUsize::new(repetitions));
                for _ in 0..repetitions {
                    deque.borrow_mut().lock().unwrap().push_back(Task::new(task.clone(), request_declaration, counter_left_tasks.clone()));
                }
            });
            tasks_counter.fetch_add(repetitions,Relaxed);
        }));
}
```

Inside ```feed_and_execute``` we will now check if the stolen task can really be stolen and if that's not the case put it back to its original deque.
```rust
fn feed_and_execute(global_queue: Shared<VecDeque<Box<dyn Fn() + Send + 'static>>>, local_deques: Vec<LocalDeque>, tasks_counter: Arc<AtomicUsize>, local_index: usize) {
    let mut rng = thread_rng();
    loop {
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
                    let task = option_task.unwrap();
                    // if the task is stealable we execute it
                    if task.is_stealable() {
                        task.execute();
                    }
                    // otherwise we put it back to its originated deque
                    else {
                        local_deques[index].lock().unwrap().push_back(task);
                    }
                }
            }
            else {
                let option_request = global_queue.lock().unwrap().pop_front();
                if option_request.is_some() {
                    option_request.unwrap()();
                }
            }
        } else {
            let option_task = local_deques[local_index].lock().unwrap().pop_back();
            if option_task.is_some() {
                option_task.unwrap().execute();
            }
        }
    }
}
```

We are left with the ```Task```'s methods to define.

For ```is_stealable```, we compare the elapsed time since ```request_declaration``` and if this time is greater or equal to a constant ```TARGET_LATENCY```, declared in our code like so : ```const TARGET_LATENCY: Duration = Duration::from_secs(4);```, then we decrement ```tasks_counter``` (passed as argument) of ```counter_left_tasks```. Doing so we are not blocked by these tasks in ```feed_and_execute```. We set to 0 ```counter_left_tasks``` and we output false. Otherwise, we simply output true.
```rust
pub fn is_stealable(&self, tasks_counter: Arc<AtomicUsize>) -> bool {
    if self.request_declaration.elapsed() >= TARGET_LATENCY {
        tasks_counter.fetch_sub(self.counter_left_tasks.load(Relaxed),Relaxed);
        self.counter_left_tasks.store(0,Relaxed);
        false
    } else {
        true
    }
}
```

For ```execute``` we simply perform the task itself and we decrement by 1 ```counter_left_tasks``` and ```tasks_counter``` passed as argument, if ```counter_left_tasks``` is not already set to 0.
```rust
pub fn execute(self, tasks_counter: Arc<AtomicUsize>) {
    (self.inner_task)();
    if self.counter_left_tasks.load(Relaxed) != 0 {
        self.counter_left_tasks.fetch_sub(1, Relaxed);
        tasks_counter.fetch_sub(1, Relaxed);
    }
}
```

Our final code :
```rust
use std::boxed::Box;
use std::cell::RefCell;
use std::collections::VecDeque;
use std::sync::{Arc, Mutex, atomic::{Ordering::Relaxed,AtomicUsize}};
use std::thread::{sleep, spawn, JoinHandle};
use rand::{thread_rng, Rng};
use std::time::{Instant,Duration};

const TARGET_LATENCY: Duration = Duration::from_secs(4);

type Shared<T> = Arc<Mutex<T>>;
type LocalDeque = Arc<Mutex<VecDeque<Task>>>;

thread_local! {
    static local_deque: RefCell<LocalDeque> =
        RefCell::new(Arc::new(Mutex::new(VecDeque::new())));
}

pub struct Task {
    inner_task : Box<dyn Fn() + Send + 'static>,
    request_declaration : Instant,
    counter_left_tasks : Arc<AtomicUsize>,
}

impl Task {
    pub fn new<T: Fn() + Send + 'static>(inner_task: T, request_declaration: Instant, counter_left_tasks: Arc<AtomicUsize>) -> Self {
        Self {
            inner_task: Box::new(inner_task),
            request_declaration,
            counter_left_tasks,
        }
    }

    pub fn execute(self, tasks_counter: Arc<AtomicUsize>) {
        (self.inner_task)();
        if self.counter_left_tasks.load(Relaxed) != 0 {
            self.counter_left_tasks.fetch_sub(1, Relaxed);
            tasks_counter.fetch_sub(1, Relaxed);
        }
    }

    pub fn is_stealable(&self, tasks_counter: Arc<AtomicUsize>) -> bool {
        if self.request_declaration.elapsed() >= TARGET_LATENCY {
            tasks_counter.fetch_sub(self.counter_left_tasks.load(Relaxed),Relaxed);
            self.counter_left_tasks.store(0,Relaxed);
            false
        } else {
            true
        }
    }
}

fn feed_and_execute(global_queue: Shared<VecDeque<Box<dyn Fn() + Send + 'static>>>, local_deques: Vec<LocalDeque>, tasks_counter: Arc<AtomicUsize>, local_index: usize) {
    let mut rng = thread_rng();
    loop {
        // if the local_deque is empty then we accept a new request from
        // the global queue OR we steal a task from another deque
        let local_is_empty = local_deques[local_index].lock().unwrap().is_empty();
        let for_exec_tasks_counter = tasks_counter.clone();
        if local_is_empty {
            // there are tasks in the system that we can steal
            if tasks_counter.load(Relaxed)!=0 {
                let tasks_counter = tasks_counter.clone();
                // we pick a random target
                let mut index;
                loop {
                    index = rng.gen::<usize>()%local_deques.len();
                    if index!=local_index {break;}
                }
                let option_task = local_deques[index].lock().unwrap().pop_back();
                if option_task.is_some() {
                    let task = option_task.unwrap();
                    // if the task is stealable we execute it
                    if task.is_stealable(tasks_counter) {
                        task.execute(for_exec_tasks_counter);
                    }
                    // otherwise we put it back to its originated deque
                    else {
                        local_deques[index].lock().unwrap().push_back(task);
                    }
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
                option_task.unwrap().execute(for_exec_tasks_counter);
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
        let request_declaration = Instant::now();
        self.global_queue
            .lock()
            .unwrap()
            .push_back(Box::new(move || {
                local_deque.with(|deque| {
                    let counter_left_tasks = Arc::new(AtomicUsize::new(repetitions));
                    for _ in 0..repetitions {
                        deque.borrow_mut().lock().unwrap().push_back(Task::new(task.clone(), request_declaration, counter_left_tasks.clone()));
                    }
                });
                tasks_counter.fetch_add(repetitions,Relaxed);
            }));
    }
}
```

With the following ```main```
```rust
fn main() {
    let mut threadpool = Threadpool::new(4);
    threadpool.forall(10, || sleep(std::time:uration::from_secs(2)));
    sleep(std::time:uration::from_secs(3));
    threadpool.forall(1, || sleep(std::time:uration::from_secs(2)));
    threadpool.forall(2, || sleep(std::time:uration::from_secs(2)));
    sleep(std::time:uration::from_secs(8));
}
```
and adding prints to see which thread is executing thanks to ```local_index```
```rust
println!("{:?} - stolen task performed in thread {}", std::time::Instant::now(), local_index);
```

we get this output
```console
PS C:\X\truct> cargo run
   Compiling truct v0.1.0 (C:\X\truct)
    Finished dev [unoptimized + debuginfo] target(s) in 0.56s
     Running `target\debug\truct.exe`
Instant { t: 177503.7388398s } - local task performed in thread 1
Instant { t: 177503.7390174s } - stolen task performed in thread 3
Instant { t: 177503.7390187s } - stolen task performed in thread 2
Instant { t: 177503.7390235s } - stolen task performed in thread 0
Instant { t: 177505.7516627s } - local task performed in thread 1
Instant { t: 177505.7516647s } - stolen task performed in thread 2
Instant { t: 177505.7516658s } - stolen task performed in thread 0
Instant { t: 177505.7516647s } - stolen task performed in thread 3
Instant { t: 177507.7570452s } - local task performed in thread 1
Instant { t: 177507.7570563s } - stolen task performed in thread 2
Instant { t: 177507.7570711s } - stolen task performed in thread 3
Instant { t: 177507.7570736s } - local task performed in thread 0
Instant { t: 177509.7667705s } - local task performed in thread 1
```
Analyzing this output, we see that thread number 1 is the one ingesting the request. Its deque is so filled with the 10 tasks. At step t = 177503 the request is so ingested by thread 1 then thread 1 execute one of its task and the other threads, 0, 2 and 3 steal a task from thread 1 and execute it. 2 seconds elapse. At step t = 177505 the thread 1 picks again a task from its local deque and the others steal a task from thread 1. All the threads execute their dedicated task and during the execution the 2 new requests are declared. From now on, tasks from the first request can't be stolen anymore given the target latency and time elapsed since request's declaration. Inside the system there are :
* 2 tasks from first request in thread 1's deque, which can't be stolen
* 2 requests inside the global queue

At step t = 177507 thread 1 executes a task from its deque and the other threads share tasks from new requests, executing them in approximately 3 seconds, so less than the target latency !
At step t = 177509 thread 1 performs its last task and we are done

