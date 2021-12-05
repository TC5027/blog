# TARGET_LATENCY_0

Consider requests presenting a high degree of inner-parallelism, for example a **forall** operation. We have resources available on which we can share our requests and their inner-parallelism.

We want to guarantee to as many requests as possible that they are entirely executed in less than **T** seconds, starting from their declaration inside the system. A first policy to share the work is to process following *work-stealing*.

Work-stealing refers to a scheduling policy aiming at minimizing time of execution of a request.
We consider **workers** and a request which can be divided into several tasks which can be executed in parallel.
Each worker has a **deque** attached which corresponds to a data structure holding tasks to complete, with both extremities reachable.
Working on a request, or a task, the worker may spawn new tasks that will be stored inside its deque in a LIFO fashion (example : Rayon's [join](https://docs.rs/rayon/1.5.1/rayon/fn.join.html) operation). A worker with an empty deque will try to feed itself by selecting a target-worker and stealing from its deque a task (if it's not empty).
The advantage of this scheduling policy is that the job of feeding the empty workers is done by these same workers !

Now, in our context where we have not one but a stream of requests to handle, we have to adapt our policy in order to regulate beginning of request's processing. A possible policy, **steal-first**, consists of authorizing a worker to process a new request only if there is no more tasks in the whole system.

With this policy, we can imagine a HUGE request, which would miss target latency anyway, occupying all resources like it could be the case if we apply steal-first. Then all requests declared after this huge one would have to wait the completion before beginning to be processed. These requests accumulate so a delay, unnecessarily as the huge request will miss target latency.

What we would like to do is having a finer control on tasks stealing by deactivating it if the goal for the initial request is already missed. Doing so, we would let worker empty, which, without tasks to steal in the system, would start process new requests.

We will set this up at the level of a **threadpool**, where the workers will be threads.

The framework in which we are going to place ourselves is as follows: 
```rust
use std::thread::{spawn,JoinHandle};

fn feed_and_execute() {
	todo!()
}

struct Threadpool {
	handlers: Vec<JoinHandle<()>>,
}

impl Threadpool {
	pub fn new(number_of_threads: usize) -> Self {
		let handlers = (0..number_of_threads).map(|_| {
			spawn(|| feed_and_execute())
		}).collect();
		Threadpool { handlers }
	}

	pub fn forall(&mut self) {
		todo!()
	}
}
```

We therefore consider a threadpool, with a ```forall``` method, which corresponds to a request, and threads all following the same policy defined by the ```feed_and_execute``` function. We will gradually complete these elements to achieve our goal.
