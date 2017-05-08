# Spring Data Reactive Demo

A very small demo Application showing various aspects of Spring Data reactive support. You can walk through the commits one by one to see the application evolve.

## What's it all about

The application is supposed to store, retrieve and expose ``Person``s from an embedded MongoDB instance in a non blocking way.

## Step 1: Setup some test data

Nothing fancy in here - we just aim to create a `Stream` of `Person` objects randomly named after characters from [Game of Thrones](http://www.georgerrmartin.com/).

```java
String[] names = { "Eddard", "Catelyn", "Jon", "Rob", "Sansa", "Aria", "Bran", "Rickon" };

Flux<Person> starks = Flux
    .fromStream(Stream.generate(() -> names[ramdom.nextInt(names.length - 1)]).map(Person::new));
```

## Step 2:

Set up a `ReactiveCrudRepository` for MongoDB and store a new `Person` from the `Stream` each and every second.

```java
Flux.interval(Duration.ofSeconds(1))
	.zipWith(starks)
	.map(Tuple2::getT2)
	.flatMap(repository::save)
	.subscribe();

interface PersonRepository extends ReactiveCrudRepository<Person, String> {}
```

## Step 3:

Create a simple derived finder method and expose the result via Spring WebFlux.

```java
@RestController
@RequiredArgsConstructor
static class PersonController {

	final PersonRepository repository;

	@GetMapping("/") // curl localhost:8080/?name=Eddard
	Flux<Person> fluxPersons(String name) {
		return repository.findAllByName(name);
	}
}

interface PersonRepository extends ReactiveCrudRepository<Person, String> {

	Flux<Person> findAllByName(String name);
}
```

## Step 4

Use RxJava `Observable` instead of Reactor `Flux` for reading and exposing data.

```java
@GetMapping("/rx") // curl localhost:8080/rx?name=Eddard
Observable<Person> rxPersons(String name) {
	return repository.findByName(name);
}

interface PersonRepository extends ReactiveCrudRepository<Person, String> {

	//...

	Observable<Person> findByName(String name);
}
```

# Step 5

Use MongoDB capped collections and tailable cursors to create an infinite stream.

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE) // curl localhost:8080/stream
Flux<Person> streamPersons() {
	return repository.findBy();
}

interface PersonRepository extends ReactiveCrudRepository<Person, String> {

	//...

	@Tailable
	Flux<Person> findBy();
}
```