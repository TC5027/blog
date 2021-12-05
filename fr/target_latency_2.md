# TARGET_LATENCY_2

La politique de work stealing s'appuie sur une deque de tâches attachée à chaque thread de la threadpool. Pour faire ça on peut utiliser la macro ```thread_local!``` comme suit : 
```rust
thread_local!{
	static local_deque: RefCell<VecDeque<Box<dyn Fn()+Send+'static>>> = 
		RefCell::new(VecDeque::new());
}
```
On peut maintenant changer le code de ```forall``` afin de découpler nos requêtes en tâches. Nos requêtes consistent à répéter X fois une tâche et donc une requête traitée par un thread doit rendre compte de ce parallélisme au niveau de la deque locale au thread en la remplissant avec ces X tâches plutot que toutes les éxécuter directement comme c'est le cas à présent. On change donc le code du forall en :
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
Sur les tasks, étant donné la répétition, on a ajouté le trait ```Clone``` pour le paramètre ```T``` à ```forall``` ce qui fait sens vu le cadre qu'on a mis en place.

Les threads doivent donc maintenant s'approvisionner depuis la queue globale **et** depuis la locale. On veut donc faire évoluer ```feed_and_execute``` afin d'intégrer le méchanisme de work-stealing qu'on a décrit en introduction avec *steal-first*.
Nos threads doivent d'abord vérifier si leur deque locale est vide ou non. Si elle contient une tâche, on la récupère et on l'effectue. Si ce n'est pas le cas alors, s'il existe des tâches dans le système, dans une des deques locales des autres threads, on essaie de trouver cette tâche et de la voler pour l'effectuer. S'il n'y a pas de tâche dans le système on prélève une requête depuis la queue globale.

Les deques locales doivent aussi etre accessibles depuis les autres threads... On va donc d'abord changer le type d'une local_deque en englobant le précédent type contenu dans la ```RefCell``` dans un ```Arc<Mutex<..>>```, afin d'avoir nos local_deque accessibles depuis les différents threads. On crée un alias pour ce type : 
```rust
type LocalDeque = Arc<Mutex<VecDeque<Box<dyn Fn()+Send+'static>>>>
```

```feed_and_execute``` va donc prendre un nouvel argument qui correspond à un ```Vec``` des local_deques présentes dans le systeme. On change le constructeur de ```Threadpool``` en initialisant les local_deques avant la création des threads. On a donc un ```Vec<Arc<Mutex<Deque<..>>>>``` qu'on clone pour chaque thread et qu'on passe en argument de feed_and_execute à l'intérieur du thread :

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

La logique à l'intérieur de ```feed_and_execute``` évolue également afin de refléter le comportement décrit au dessus. On a besoin de  garder une trace de la présence de tâches dans le système, ce que l'on fait à l'aide d'un compteur global ```tasks_counter: Arc<AtomicUsize>``` initialisé à 0.

```rust
struct Threadpool {
    handlers: Vec<JoinHandle<()>>,
    global_queue: Shared<VecDeque<Box<dyn Fn() + Send + 'static>>>,
    tasks_counter: Arc<AtomicUsize>,
}
```

Ce compteur est ensuite manipulé dans le code de forall. A l'initialisation d'une requête, on incrémente de ```repetitions``` et à chaque task accomplie, on décrémente de 1.

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

Finalement, on a tout ce qu'il nous faut pour implémenter la logique dans ```feed_and_execute```

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

A ce moment, notre code ressemble à cela : (rand = "0.8.0")
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

On peut faire une petite démonstration pour démontrer qu'on a bien le comportement attendu avec le ```main``` suivant : 

```rust
fn  main() {
	let  mut  threadpool = Threadpool::new(4);
	threadpool.forall(8, || sleep(std::time:uration::from_secs(2)));
	sleep(std::time:uration::from_secs(5));
}
```

et en ayant ajouté des ```println!``` dans ```feed_and_execute``` au niveau de l'éxécution des tâches,
```rust
println!("{:?} - stolen/local task performed", std::time::Instant::now());
option_task.unwrap()();
```
on obtient l'output suivant :

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

On a bien une mise en évidence du work-stealing, avec les tâches qui se retrouvent réparties sur les différents threads. Avec 4 threads dans la threadpool on effectue l'ensemble de la requête, qui se découple en 8 tasks, en 2 "passes" si l'on prête attention aux timings.

On peut de même illustrer le phénomène néfaste qu'on décrivait en intro si on ajoute avant la boucle ```for``` dans ```forall``` : 
```rust
deque.borrow_mut().lock().unwrap().push_back(Box::new(move || {
    println!("it took {:?} to execute the request since entering the system", enter_time.elapsed());
}));
```

Notre précédent code produit maintenant l'output suivant  :
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

Si on considère le ```main``` suivant, en se disant que la target latency est de 5 secondes :
```rust
fn  main() {
	let  mut  threadpool = Threadpool::new(4);
	threadpool.forall(20, || sleep(std::time:uration::from_secs(2)));
    sleep(std::time:uration::from_secs(1));
    threadpool.forall(4, || sleep(std::time:uration::from_secs(2)));
	sleep(std::time:uration::from_secs(60));
}
```

On a l'output suivant :

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

La première requête (qui allait obligatoirement louper la target_latency de 5 secondes) a monopolisé toutes les ressources le temps de son éxécution et la seconde requête, qui aurait pu atteindre la target_latency, loupe donc également.

