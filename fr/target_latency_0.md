# TARGET_LATENCY_0

On considère des requêtes présentant un haut degré de parallélisme, par exemple une opération **forall**. On a des ressources disponibles, sur lesquelles on peut répartir nos requêtes et leur parallélisme.

On veut garantir au plus grand nombre de requêtes possible que la requête est entièrement éxécutée en moins de **T** secondes à partir de sa déclaration dans le système. Une première politique pour répartir le travail est de procéder en prenant une approche *work-stealing*.

Le work-stealing fait référence à une politique d'ordonnancement visant à minimiser le temps de complétion d'une requête.
On considère des **workers** et une requête qui peut etre décomposée en plusieurs tâches pouvant etre éxécutées en parallèle.
Chaque worker a une **deque** attachée qui correspond à une structure de données, stockant les tâches à compléter, avec les deux extrémités qui sont accessibles.
En traitant la requête, ou une tâche, le worker va possiblement faire apparaitre de nouvelles tâches, que le worker stocke dans sa deque à la manière d'une pile (exemple : opération [join](https://docs.rs/rayon/1.5.1/rayon/fn.join.html) de Rayon). Un worker qui a une deque vide va essayer de s'approvisionner en choisissant un worker-cible et en lui volant une tâche de sa deque (si elle n'est pas vide). 
L'avantage de cette politique d'ordonnancement est que le travail qui consiste à approvisionner les workers en attente est fait par ces workers eux-mêmes !

Maintenant, dans notre contexte où l'on n'a pas une seule requête à traiter mais tout un stream, on doit adapter cette politique afin de réguler le début du traitement des requêtes. Une politique possible, **steal-first**, est d'authoriser un worker à traiter une nouvelle requête seulement lorsqu'il n'y a plus aucune tâche dans toutes les deques du système.

Avec cette politique, si on imagine une ENORME requête, qui dans tous les cas louperait l'objectif, monopolisant l'ensemble des ressources comme c'est le cas si on applique steal-first, alors toutes les requêtes déclarées ensuite, potentiellement plus légères, vont devoir attendre que l'enorme requête soit entièrement traitée avant de commencer à être traitées par le système. Les requêtes déclarées pendant que l'énorme est en traitement cumulent donc du retard, et inutilement puisque l'énorme allait quoi qu'il arrive louper l'objectif.

On voudrait donc avoir un controle plus fin sur le stealing de tâches en le rendant infaisable si l'objectif pour la requête d'origine est de toutes façons déja loupé. Cela laisserait donc des workers vides, qui, n'ayant pas de tâches pouvant etre volés à disposition, pourront commencer à traiter une nouvelle requête.

On va mettre cela en place au niveau d'une **threadpool**, où les workers seront des threads.

Le cadre dans lequel on va se placer est le suivant : 
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


On considère donc une threadpool, avec une méthode ```forall```, qui correspond à une requête, et des threads suivant tous la même politique définie par la fonction ```feed_and_execute```. Nous allons au fur et à mesure compléter ces éléments pour parvenir à notre objectif.
