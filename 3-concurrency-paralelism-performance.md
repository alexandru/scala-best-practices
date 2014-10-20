<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" />

## 3. Concurrency, Parallelism and Performance

### 3.1. SHOULD avoid concurrency like the plague it is

Avoid having to deal with concurrency as much as possible. People good
at concurrency avoid it like the plague it is.

**WARNING:** concurrency issues happen not only when speaking about
shared memory and threads, but also between processes when contention
on any kind of resource (like a database) is involved.

Example - when a job is scheduled to execute every minute by using
cron.d in Linux and that job fetches and updates items from a queue
persisted in MySQL, that job can take longer than 1 minute to execute
and thus you can end up with 2 or 3 processes executing at the same
time and contending on the same MySQL table.

### 3.2. SHOULD use appropriate abstractions only where suitable - Future, Actors, Rx

Learn about the abstractions available and choose between them
depending on the task at hand. There is no silver bullet that can be
generally applied. The more high-level the abstraction, the less scope
it has in solving issues. For example many developers in the Scala
community are overusing Akka Actors - which are great, but not when
misapplied. Like don't use an Akka Actor when a `Future` would do.

Scala's Futures and Promises are good because:

- they are inherently parallelizable by eliminating concurrency
  concerns
- fairly efficient, because when submitting a task to the implicit
  `ExecutionContext`, this execution context efficiently multiplexes
  between few threads by default (the number of threads in the
  thread-pool is often directly proportional to the number of CPU cores
  you have)
- the model is inherently simple and easy to use

Futures and Promises are bad because they signal only one value from
the producer to the consumer and that's it - if you need a stream or
bi-directional communications, a Future might not be the best
abstraction.

Akka Actors are good because:

- they make bidirectional communications over asynchronous boundaries
  easy - for example WebSocket is a prime candidate for actors
- with Akka's Actors you can easily model state machines (see
  `context.become`)
- the processing of messages has a strong guarantee of
  non-concurrency - messages are processed one by one, so there's no
  need to worry about concurrency issues while in the context of an
  actor
- instead of implementing a half-assed in-memory queue for processing
  of things, you could just use an actor, since queuing of messages and
  acting on those messages is what they do

Akka's Actors are bad because:

- they are fairly low-level for many tasks
- it's extremely easy to model actors that keep a lot of state, ending
  up with a system that can't be horizontally scaled
- because of the bidirectional communications capability, it's
  extremely easy to end up with data flows that are so complex as to be
  unmanageable
- the model in general is actor A sending a message to actor B - but
  if you need to model a stream of events, this tight coupling between A
  and B is not acceptable

Rx and Iteratees - see Play's
[Iteratees](https://www.playframework.com/documentation/2.3.x/Iteratees)
/ [RxJava](https://github.com/ReactiveX/RxJava) /
[Reactive Streams](http://www.reactive-streams.org/) /
[Monifu](https://github.com/alexandru/monifu) are good because:

- they model unidirectional communications between a producer and
  consumers - with an `Observable`/`Enumerator` being a channel that doesn't
  care what listeners it has (think about a stream of data like a river
  of information - the river doesn't care about who drinks from it)
- depending on implementation, they address back-pressure concerns by
  default, with consumers signaling demand to the producer and the
  producer insuring that it doesn't send more data than the consumer can
  chew on
- just as in the case of Future, because of the limitations, the model
  is simple to use - and the exposed operators / combinators are awesome

Rx / Iteratees are bad because:

- they are only about unidirectional communications, it gets
  complicated if you want bidirectional communications, so choose
  actors instead

- because of the strong contract they come with (i.e. no concurrent
  notifications for example), implementing new operators and data-source
  can be problematic, but usage on the consumer side is kept simple
  because of this

So there you have it. Learn and pick wisely - don't apply abstractions
like some sort of special sauce without thinking about it

### 3.3. MUST NOT wrap purely CPU-bound operations in Futures

This is in general an anti-pattern:

```scala
def add(x: Int, y: Int) = Future { x + y }
```

If you don't see any kind of I/O in there, then that's a red
flag. Shoving stuff in Futures without thinking is not going to solve
your performance problems. Especially in the case of a web server, in
which the requests are already paralellized and the above would get
executed in response to requests, shoving purely CPU-bound in that
Future constructor will make your logic slower to execute, not faster.

### 3.4. MUST use Scala's BlockContext on blocking I/O

This includes all blocking I/O, including SQL queries. Real sample:

```scala
Future {
  DB.withConnection { implicit connection =>
    val query = SQL("select * from bar")
    query()
  }
}
```

Blocking calls are error-prone because one has to be aware of exactly
what thread-pool gets affected and given the default configuration of
the backend app, this can lead to non-deterministic dead-locks. It's a
bug waiting to happen in production.

Here's a simplified example demonstrating the issue for didactical purposes:

```scala
implicit val ec = ExecutionContext
  .fromExecutor(Executors.newFixedThreadPool(1))

def addOne(x: Int) = Future(x + 1)

def multiply(x: Int, y: Int) = Future {
  val a = addOne(x)
  val b = addOne(y)
  val result = for (r1 <- a; r2 <- b) yield r1 * r2

  // this will dead-lock
  Await.result(result, Duration.Inf)
}
```

This sample is simplified to make the effect deterministic, but all
thread-pools configured with upper bounds will sooner or later be
affected by this.

Blocking calls have to be marked with a `blocking` call that signals
to the `BlockContext` a blocking operation. It's a very neat mechanism
in Scala that lets the `ExecutionContext` know that a blocking operation
happens, such that the `ExecutionContext` can decide what to do about
it, such as adding more threads to the thread-pool (which is what
Scala's ForkJoin thread-pool does).

All the code has to be reviewed and whenever a blocking call happens, this is the fix:

```scala
import scala.concurrent.blocking
// ...
blocking {
  someBlockingCallHere()
}
```

### 3.5. SHOULD NOT block

Sometimes you have to block the thread underneath - unfortunately JDBC
doesn't have a non-blocking API. However, when you have a choice,
never, ever block. For example, don't do this:

```scala
def fetchSomething: Future[String] = ???

// later ...
val result = Await.result(fetchSomething, 3.seconds)
result.toUpperCase
```

Prefer keeping the context of that Future all the way:

```scala
def fetchSomething: Future[String] = ???

fetchSomething.map(_.toUpperCase)
```

Also checkout [Scala-Async](https://github.com/scala/async) to make
this easier.

### 3.6. All public APIs SHOULD BE thread-safe

As a general rule of software engineering on top of the JVM,
absolutely all public APIs from inside your process will end up being
used in a context in which multiple threads are using these APIs at
the same time. It's best practice for all public APIs (all components
in your Cake for example) to be designed as thread-safe from the get
go, because you do not want to chase concurrency bugs.

And if a public API is not thread-safe for some reason (like the usual
compromises made in software development), then state this fact in
BOLD CAPITAL LETTERS.

The reason is simple - if an API is not thread-safe, then there's no
way a user of that API can know about the way to synchronize on
it. Example:

```scala
val list = mutable.List.empty[String]
```

Lets say this mutable list is exposed. On what lock is a user supposed
to synchronize? On the list itself? That's not enough, being a pretty
useless lock to have in a larger context. Remember, locks are not
composable and are very error-prone. Never leave the responsibility of
synchronizing for contention on your users.

### 3.7. SHOULD avoid contention on shared reads

Meet
[Amdahl's Law](https://en.wikipedia.org/wiki/Amdahl's_law). Synchronizing
with locks drastically limits the parallelization possible and thus
vertical scalability. Reads are embarrassingly paralellizable, so
avoid at all times doing this:

```scala
def fetch = synchronized { someValue }
```

Come up with better synchronization schemes that does not involve
synchronizing reads, like atomic references or STM. If you aren't able
to do that, then avoid this altogether by using proper abstractions.

### 3.8. MUST evolve the state of actors only in response to messages received from the outside

When using Akka actors, their mutable state should always evolve in
response to messages received from the outside. An anti-pattern that
comes up a lot is this:

```scala
class SomeActor extends Actor {
  private var counter = 0
  private val scheduler = context.system.scheduler
    .schedule(3.seconds, 3.seconds, self, Tick)

  def receive = {
    case Tick =>
      counter += 1
  }
}
```

In the example above the actor schedules a Tick every 3 seconds that
evolves its state. This is an extremely costly mistake. The actor's
behavior becomes totally non-deterministic and impossible to test
right.

If you really need to periodically do something inside an actor, then
that scheduler must not be initialized inside the actor. Take it out.

### 3.9. MUST provide a clearly defined and documented protocol for each component or actor that communicates over async boundaries

A function signature is not enough for documenting the protocol of
problematic components. Especially when talking about communications
over asynchronous boundaries (between threads, between processes on
the network, etc...), the protocol needs a lot of detail on what you
can and cannot rely on.

As a guideline, don't shy away from writing comments and document:

- concurrency and latency concerns
- the proper ordering of calls
- everything that can go wrong

### 3.10. SHOULD avoid having mutable state in persistent actors

If you keep mutable state inside your actors, it makes it harder to
parallelize the workload - actors with a lot of mutable state in them
tend to have problems with horizontal scalability. Always take
horizontal scalability into account and ask yourself - is this
scalable? Could multiple actors of this type run in parallel?

### 3.11. SHOULD always prefer single-producer scenarios

Shared writes are not parallelizable, whereas shared reads are
embarrassingly parallelizable. As a metaphor, 100,000 people can watch
the same soccer game on the same stadium at the same time (reads), but
100,000 people cannot all use the same bathroom (writes). In a
multi-threading scenario, prefer single producer / multi consumer
scenarios. This has the effect of avoiding contention and performance
problems. An app does not scale vertically with multiple producers
pounding on the same resource, because Amdahl's Law.

Checkout [LMAX Disruptor](https://lmax-exchange.github.io/disruptor/).

### 3.12. MUST be mindful of the garbage collector and how unnecessary allocations can kill performance

Don't over allocate resources, unless you need to. We want to avoid
micro optimizations, but always be mindful about the effects
allocations can have on your system.

In the
[words of Martin Thomson](http://www.infoq.com/presentations/top-10-performance-myths),
if you stress the garbage collector, you'll increase the latency on
stop-the-world freezes and the number of such occurrences, with the
garbage collector acting like a GIL and thus limiting performance and
vertical scalability.

Example:

```scala
query.filter(_.someField.inSet(Set(name)))
```

This is a sample that occurred in our project due to a problem with
Slick's API. So instead of a `===` test, the developer chose to do an
`inSet` operation with a sequence of 1 element. This allocation of a
collection of 1 element happens on every method call. Now that's not
good, what can be avoided should be avoided.

Another example:

```scala
someCollection
 .filter(x => Set(a,b,c).contains(x.id))
 .map(_.name)
```

First of all, this creates a Set every single time, on each element of
our collection. Second of all, filter and map can be compressed in one
operation, otherwise we end up with more garbage and more time spent
building the final collection:

```scala
val validIDs = Set(a,b,c)

someCollection.collect {
 case x if validIDs.contains(x.id) =>
   x.name
}
```

A generic example that often pops up, exemplifying useless traversals
and operators that could be compressed:

```scala
collection.filter(bySomething).map(toSomethingElse).filter(again).headOption
```

Also, take notice of your requirements and use the data-structure
suitable for your use-case. You want to build a stack? That's a
`List`. You want to index a list? That's a `Vector`. You want to
append to the end of a list? That's again a `Vector`. You want to push
to the front and pull from the back? That's a `Queue`. You have a set
of things and want to check for membership? That's a `Set`. You have a
list of things that you want to keep ordered? That's a
`SortedSet`. This isn't rocket science, just computer science 101.

We are not talking about extreme micro optimizations here, we aren't
even talking about something that's Scala, or FP, or JVM specific
here, but be mindful of what you're doing and try to not do
unnecessary allocations, as it's much harder fixing it later.

### 3.13. MUST NOT hardcode the thread-pool / execution context

This is a general design issue related to the project as a whole, but don't do this:

```scala
import scala.concurrent.ExecutionContext.Implicits.global

def doSomething: Future[String] = ???
```

Tight coupling between the execution context and your logic is not
good and that import is tight-coupling, especially since in the
context of a Play2 application you need to use a
[different thread-pool](https://www.playframework.com/documentation/2.3.x/ThreadPools).

Just pass the ExecutionContext around as an implicit parameter. It's
idiomatic and acceptable this way:

```scala
def doSomething(implicit ec: ExecutionContext): Future[String] = ???
```
