## 5. Akka Actors

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 5.1. SHOULD evolve the state of actors only in response to messages received from the outside

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

### 5.2. SHOULD mutate state in actors only with context.become

Say we've got an actor that mutates its state (most actors do),
doesn't even matter what state that is:

```scala
class MyActor extends Actor {
  val isInSet = mutable.Set.empty[String]

  def receive = {
    case Add(key) =>
      isInSet += key
      
    case Contains(key) =>
      sender() ! isInSet(key)
  }
}

// Messages
case class Add(key: String)
case class Contains(key: String)
```

Since we are using Scala, we want to be as pure as practically
possible, we want to deal with immutable data-structures and pure
functions, we want to go FP to reduce the area for
[accidental complexity](https://en.wikipedia.org/wiki/No_Silver_Bullet)
and let me tell you, there's nothing pure, immutable or referentially
transparent about the above ;-)

Meet [context.become](http://doc.akka.io/docs/akka/2.3.4/scala/actors.html#Become_Unbecome):

```scala
import collection.immutable.Set

class MyActor extends Actor {
  def receive = active(Set.empty)

  def active(isInSet: Set[String]): Receive = {
    case Add(key) =>
      context become active(isInSet + key)
      
    case Contains(key) =>
      sender() ! isInSet(key)
  }
}
```

If that doesn't instantly ring a bell, just wait until you'll have to
model a state machine with 10 states in it and dozens of possible
transitions and effects to go along with it, then you'll get it.

### 5.3. MUST NOT leak the internal state of an actor in asynchronous closures

Again with the mutable state, spot the problem:

```scala
class MyActor extends Actor {
  val isInSet = mutable.Set.empty[String]

  def receive = {
    case Add(key) =>
      for (shouldAdd <- validate(key)) {
        if (shouldAdd) isInSet += key
      }
        
    // ...
  }
  
  def validate(key: String): Future[Boolean] = ???
}
```

Chaos ensues, hell's doors open for a whole range of non-deterministic
bugs that could happen due to multi-threading issues. This is a
general problem with functions that execute asynchronously and that
capture variables that aren't meant to escape their
context. [Spores](http://docs.scala-lang.org/sips/pending/spores.html)
is a proposal for macros-enabled closures that are supposed to make
this safer, but until then just be careful.

First of all, see the rule about using `context.become` for mutating
state, which is already a step in the right direction. And then you
need to deal with this by sending another message to our actor when
our future is done:

```scala
import akka.pattern.pipeTo

class MyActor extends Actor {
  val isInSet = mutable.Set.empty[String]

  def receive = {
    case Add(key) =>
      val f = for (isValid <- validate(key))
        yield Validated(key, isValid)
        
      // sending the result as a message back to our actor
      f pipeTo self

    case Validated(key, isValid) =>
      if (isValid) isInSet += key
              
    // ...
  }
  
  def validate(key: String): Future[Boolean] = ???
}

// Messages
case class Add(key: String)
case class Validated(key: String, isValid: Boolean)
```

And of course, we could be modeling a state-machine that doesn't
accept any more requests until the last one is done. Let us also get
rid of that mutable collection and also introduce back-pressure
(i.e. we need to tell the sender when it can send the next item):

```scala
import akka.pattern.pipeTo

class MyActor extends Actor {
  def receive = idle(Set.empty)

  def idle(isInSet: Set[String]): Receive = {
    case Add(key) =>
      // sending the result as a message back to our actor
      validate(key).map(Validated(key, _)).pipeTo(self)
      
      // waiting for validation
      context.become(waitForValidation(isInSet, sender()))
  }

  def waitForValidation(set: Set[String], source: ActorRef): Receive = {
    case Validated(key, isValid) =>
      val newSet = if (isValid) set + key else set
      // sending acknowledgement of completion
      source ! Continue
      // go back to idle, accepting new requests
      context.become(idle(newSet))

    case Add(key) =>
      sender() ! Rejected
  }

  def validate(key: String): Future[Boolean] = ???
}

// Messages

case class Add(key: String)
case class Validated(key: String, isValid: Boolean)
case object Continue
case object Rejected
```

Yeap, actor-based designs can get tricky.

### 5.4. SHOULD do back-pressure

Say you've got an actor that produces values - like reading items from
a RabbitMQ or your own half-assed queue stored in a MySQL table, or
files that have to be observed and processed as soon as the actor sees
them popping up in a certain directory and so on. This producer needs
to push work into a number of variable actors.

Problems:

1. if the queue of messages is unbounded, with slow consumers that
   queue can blow up
2. distribution can be inefficient, as a worker could end up with
   multiple pending items whereas another worker could be standing
   still

A correct, worry-free design does this:

- workers must signal demand (i.e. when they are ready for processing more items)
- the producer must produce items only when there is demand from workers

Here's a detailed sample with comments:

```scala
/**
 * Message signifying acknowledgement that upstream can send the next
 * item.
 */
case object Continue

/**
 * Message used by the producer for continuously polling the
 * data-source, while in the polling state.
 */
case object PollTick

/**
 * State machine with 2 states:
 *
 *  - Standby, which means there probably is a pending queue of items waiting to
 *    be sent downstream, but the actor is waiting for demand to be signaled
 * 
 *  - Polling, which means that there is demand from downstream, but the
 *    actor is waiting for items to happen
 *
 * IMPORTANT: as a matter of protocol, this actor must not receive multiple
 *            Continue events - downstream Router should wait for an item
 *            to be delivered before sending the next Continue event to this
 *            actor.
 */
class Producer(source: DataSource, router: ActorRef) extends Actor {
  import Producer.PollTick

  override def preStart(): Unit = {
    super.preStart()
    // this is ignoring another rule I care about (actors should evolve
    // only in response to external messages), but we'll let that be
    // for didactical purposes
    context.system.scheduler.schedule(1.second, 1.second, self, PollTick)
  }

  // actor starts in standby state
  def receive = standby

  def standby: Receive = {
    case PollTick =>
      // ignore

    case Continue =>
      // demand signaled, so try to send the next item
      source.next() match {
        case None =>
          // no items available, go in polling mode
          context.become(polling)
          
        case Some(item) =>
          // item available, send it downstream,
          // and stay in standby state
          router ! item
      }
  }

  def polling: Receive = {
    case PollTick =>
      source.next() match {
        case None =>
          () // ignore - stays in polling
        case Some(item) =>
          // item available, demand available
          router ! item
          // go in standby
          context.become(standby)
      }
  }
}

/**
 * The Router is the middleman between the upstream Producer and
 * the Workers, keeping track of demand (to keep the producer simpler).
 *
 * NOTE: the protocol of Producer needs to be respected - so
 *       we are signaling a Continue to the upstream Producer
 *       after and only after a item has been sent downstream
 *       for processing to a worker. 
 */
class Router(producer: ActorRef) extends Actor {
  var upstreamQueue = Queue.empty[Item]
  var downstreamQueue = Queue.empty[ActorRef]

  override def preStart(): Unit = {
    super.preStart()
    // signals initial demand to upstream
    producer ! Continue
  }

  def receive = {
    case Continue =>
      // demand signaled from downstream, if we have items to send
      // then send, otherwise enqueue the downstream consumer
      if (upstreamQueue.isEmpty) {
        downstreamQueue = downstreamQueue.enqueue(sender)
      }
      else {
        // no need to signal demand upstream, since we've got queued
        // items, just send them downstream
        val (item, newQueue) = upstreamQueue.dequeue
        upstreamQueue = newQueue
        sender ! item

        // signal demand upstream for another item
        producer ! Continue
      }

    case item: Item =>
      // item signaled from upstream, if we have queued consumers
      // then signal it downstream, otherwise enqueue it
      if (downstreamQueue.isEmpty) {
        upstreamQueue = upstreamQueue.enqueue(item)
      }
      else {
        val (consumer, newQueue) = downstreamQueue.dequeue
        downstreamQueue = newQueue
        consumer ! item

        // signal demand upstream for another item
        producer ! Continue
      }
  }
}

class Worker(router: ActorRef) extends Actor {
  override def preStart(): Unit = {
    super.preStart()
    // signals initial demand to upstream
    router ! Continue
  }
  
  def receive = {
    case item: Item =>
      process(item)
      router ! Continue
  }
}
```
