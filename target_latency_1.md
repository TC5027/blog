# TARGET_LATENCY_1

The requests we will consider are ```forall``` operations (presenting intra-parallelism). For this operation, we will limit ourselves to repeting a task X times, tasks being independent from each other. The signature becomes
```rust
pub fn forall<T: Fn()>(&mut self, repetitions : usize, task : T) 
```

This request has to be made available in a global queue dedicated to requests, defined as a field of ```Threadpool```. We use ```VecDeque``` to simulate the queue's behavior thanks to ```push_back``` and ```pop_front```.
We modify our struct into 
```rust
struct Threadpool {
	handlers: Vec<JoinHandle<()>>,
	global_queue: VecDeque<dyn Fn()>,
}
```

At compilation we find an error telling us that for ```dyn Fn()``` we have an unknown size at compilation which is forbidden by the compilator. The trick is to use ```Box``` to send this on the heap and work with a pointer of known size at compilation. We can now fill (partially) ```forall``` : 
```rust
pub fn forall<T : Fn()>(&mut self, repetitions : usize, task : T) {
	self.global_queue.push_back(Box::new(move || {
		for _ in 0..repetitions {
			task();
		}
	}));
	todo!()
}
```

We have another error on the ```T``` parameter, which risks to not live long enough. We solve this (brutally) by adding a ```'static``` lifetime to this parameter, saying our tasks will live for the entire lifetime of the running program.

Now, we want our threads to be able to access to this queue. They are governed by the function ```feed_and_execute``` and so it is at this level that we will interact with the queue. In order to do so we need to specify that our requests implement ```Send``` in order to be moved to threads. We change so the type inside our queue to ```Box<dyn Fn()+Send+'static```. Furthermore, given the fact that we have to share our queue between several threads, we encompass it inside an ```Arc<Mutex<>>```. We create an alias for this type : ```Shared```.
Inside ```feed_and_execute``` we loop on the queue and execute the requests we get from it.

The code is as following :
```rust
use std::thread::{spawn,JoinHandle};
use std::collections::VecDeque;
use std::boxed::Box;
use std::sync::{Arc,Mutex};

type Shared<T> = Arc<Mutex<T>>;

fn feed_and_execute(global_queue: Shared<VecDeque<Box<dyn Fn()+Send+'static>>>) {
	loop {
		let option_request = global_queue.lock().unwrap().pop_front();
		if option_request.is_some() {
			option_request.unwrap()();
		}
	}
}

struct Threadpool {
	handlers: Vec<JoinHandle<()>>,
	global_queue: Shared<VecDeque<Box<dyn Fn()+Send+'static>>>,
}

impl Threadpool {
	pub fn new(number_of_threads: usize) -> Self {
		let global_queue = Arc::new(Mutex::new(VecDeque::new()));
		let handlers = (0..number_of_threads).map(|_| {
			let global_queue = global_queue.clone();
			spawn(|| feed_and_execute(global_queue))
		}).collect();
		Threadpool { 
		    handlers,
		    global_queue,
		}
	}

	pub fn forall<T : Fn()+Send+'static>(&mut self, repetitions : usize, task : T) {
		self.global_queue.lock().unwrap().push_back(Box::new(move || {
		    for _ in 0..repetitions {
		        task();
		    }
		}));
		todo!()
	}
}
```
