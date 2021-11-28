# TARGET_LATENCY_3

Dans cette dernière partie on va mettre en place un méchanisme empéchant les requêtes ayant un temps total d'éxécution supérieur à la target_latency de voir leurs tâches pouvoir etre volées par un autre thread. Pour approfondir ce sujet, je recommande la lecture de ce [papier](https://www.cse.wustl.edu/~angelee/home_page/papers/steal.pdf). De cette manière on laisse des threads libres pour traiter des nouvelles requêtes qui aurait pu etre retardées par le traitement d'une requête traitée par tout le système alors qu'elle ne pouvait être accomplie dans le temps imparti.

On doit donc mettre en place un moyen de mesurer le temps d'éxécution d'une requête dispatchée en différentes tâches. Pour ce faire, on va créer une nouvelle structure dédiée aux tâches : ```Task```
Cette structure va ressembler à ça  :

```rust
pub struct Task {
    inner_task : Box<dyn Fn() + Send + 'static>,
    request_declaration : Instant,
    counter_left_tasks : Arc<AtomicUsize>,
}
```

On a donc un champ correspondant à la tâche même à éxécuter et un champ qui correspond à l'instant où la requête d'où découle la tâche a été déclarée. Le troisième champ correspond au nombre de tâches, issues de la requête, encore présentes dans le système (càd pas encore éxécutées). 

On va avoir besoin de 3 méthodes pour cette struture : 
*	un constructeur
*	une méthode pour éxécuter la tâche
*	une méthode pour savoir si la tâche peut etre volée ou pas

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

Afin d'intégrer ce méchanisme de vol controlé, on doit changer le niveau auquel s'effectue l'éxécution d'une tâche. En effet, avec le code actuel, ce qu'on récupère des deque est directement une fonction à éxécuter. Ce qu'on va faire maintenant c'est plutot remplir les deques avec des instances de ```Task``` qu'on pourra d'abord analyser puis peut-être éxécuter.

L'alias ```LocalDeque``` devient donc ```type  LocalDeque = Arc<Mutex<VecDeque<Task>>>;``` et on change le code de ```forall```.

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

Dans ```feed_and_execute``` on va maintenant contrôler le fait que la tâche qu'on vole peut effectivement l'être et si ce n'est pas le cas, la remettre dans sa deque d'origine.

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

Il ne nous reste enfin plus que les méthodes de ```Task``` à définir.

Pour ```is_stealable```, on compare le temps elapsed depuis ```request_declaration``` et si ce temps est supérieur ou égal à une constante ```TARGET_LATENCY``` déclarée dans le programme, par exemple à 4 secondes : ```const TARGET_LATENCY: Duration = Duration::from_secs(4);```, alors on décrémente le ```tasks_counter``` (passé en argument) de ```counter_left_tasks```, comme ça on est pas bloqué par ces tâches dans ```feed_and_execute```. On met à 0 ```counter_left_tasks```  et on renvoie ```false```. Dans le cas contraire on renvoie simplement ```true```.
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

Pour ```execute```, on effectue la tâche en elle-même et on décrémente de 1 le ```counter_left_tasks``` ET et le ```tasks_counter``` passé en argument si ```counter_left_tasks``` n'est pas déja à 0.
```rust
pub fn execute(self, tasks_counter: Arc<AtomicUsize>) {
    (self.inner_task)();
    if self.counter_left_tasks.load(Relaxed) != 0 {
        self.counter_left_tasks.fetch_sub(1, Relaxed);
        tasks_counter.fetch_sub(1, Relaxed);
    }
}
```

Notre code final :
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
                    deque.borrow_mut().lock().unwrap().push_back(Task::new(Box::new(move || {
                    }),request_declaration, counter_left_tasks.clone()));
                    
                    for _ in 0..repetitions {
                        deque.borrow_mut().lock().unwrap().push_back(Task::new(task.clone(), request_declaration, counter_left_tasks.clone()));
                    }
                });
                tasks_counter.fetch_add(repetitions,Relaxed);
            }));
    }
}
```

avec le main suivant
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
et la target_latency de 4 secondes en indiquant l'origine du thread grâce au ```local_index``` dans les prints
```rust
println!("{:?} - stolen task performed in thread {}", std::time::Instant::now(), local_index);
```
on a l'output suivant :

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

En analysant l'output, on voit que le thread numero 1 est celui qui ingère la requête et qui a donc sa deque remplit avec ses 10 tâches. A la passe t=177503 on a donc la requête qui est ingérée par le thread 1 puis le thread 1 éxécute une des tâches et les autres threads, 0, 2 et 3, volent une tâche au thread 1 et l'éxécutent. Il s'écoule 2 secondes. A la passe t=177505 le thread 1 repioche une tâche locale et les autres volent une tâche au thread 1, tous l'éxécutent et pendant l'éxécution on a les 2 nouvelles requêtes qui sont déclarées. A partir de maintenant, les tâches de la première requête ne peuvent plus être volées étant donnée la latence et le temps écoulé. On a donc dans le système :
* 2 tâches de la première requête dans la deque du thread 1, qui ne peuvent plus être volées
* 2 requêtes dans la queue globale
A la passe t=177507 on a donc le thread 1 qui éxécute une de ses tâches et les autres threads qui se répartissent les tâches des nouvelles requêtes, complétant ainsi ces deux requêtes en environ 3 secondes, càd en moins de temps que la latence !
A la passe t=177509 on a seulement le thread 1 qui s'occupe de sa dernière tâche restante et on a ainsi tout bouclé.

