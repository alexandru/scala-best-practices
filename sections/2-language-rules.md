## 2. Language Rules

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 2.1. MUST NOT use "return"

The `return` statement from Java signals a side-effect - unwind the
stack and give this value to the caller. In a language in which the
emphasis is on side-effect-full programming, this makes sense. However
Scala is an expression oriented language in which the emphasis is on
controlling/limiting side-effects and `return` is not idiomatic.

To make matters worse, `return` probably doesn't behave as you think
it does. For example in a Play controller, try doing this:

```scala
def action = Action { request =>
  if (someInvalidationOf(request))
    return BadRequest("bad")

 Ok("all ok")
}
```

In Scala, a `return` statement inside a nested anonymous function is
implemented by throwing and catching a `NonLocalReturnException`. It
says so in the
[Scala Language Specification, section 6.20](http://www.scala-lang.org/docu/files/ScalaReference.pdf).

Besides, `return` is anti structural programming, as functions can be
described with multiple exit points and if you need `return`, like in
those gigantic methods with lots of if/else branches, the presence of
a `return` is a clear signal that the code stinks, a magnet for future
bugs and is thus in urgent need of refactoring.

### 2.2. SHOULD use immutable data-structures

Immutable data-structures are facts that can be compared and reasoned
about. Mutable things are error-prone buckets. You should never use a
mutable data-structure unless you're able to defend it and there are
really, really few places in which a mutable data-structure is
defensible.

Lets exemplify:

```scala
trait Producer[T] {
 def fetchList: List[T]
}

// consumer side
someProducer.fetchList
```

Question - if the `List` returned above is mutable, what does that say
about the `Producer` interface?

Here are some problems:

1. if this List is produced on another thread than the consumer, one
   can have both visibility and atomicity problems - you can't know
   whether that will happen, unless you take a look at the Producer's
   implementation.

2. even if this List is effectively immutable (i.e. still mutable, but
   no longer modified after being signaled to the Consumer), you don't
   know if it will be signaled to other Consumers that may modify it
   by themselves, so you can't reason about what you can do with it.

3. even if it is described that access to this List must be
   synchronized, problem is - on which lock are you going to
   synchronize?  Are you sure you'll get the locking order right?
   Locks are not composable.

So there you have it - a public API exposing a mutable data-structure
is an abomination of nature, leading to problems that can be worse
than what happens when doing manual memory management.

### 2.3. SHOULD NOT update a `var` using loops or conditions

It's a mistake that most Java developers do when they come to Scala. Example:

```scala
var sum = 0
for (elem <- elements) {
  sum += elem.value
}
```

Avoid doing this, prefer the available operators instead, like `foldLeft`:

```scala
val sum = elements.foldLeft(0)((acc, e) => acc + e.value)
```

Or even better, know the standard library and always prefer to
use the built-in functions - the more expressive you go, the less bugs
you'll have:

```scala
val sum = elements.map(_.value).sum
```

In the same spirit, you shouldn't update a partial result with a condition.
Example:


```scala
def compute(x) = {
  var result = resultFrom(x)

  if(needToAddTwo) {
    result += 2
  }
  else {
    result += 1
  }

  result
}
```

Prefer expressions and immutability. The code will be more readable
and less error-prone, since it makes the branches more explicit and that's
a good thing:

```scala
def computeResult(x) = {
  val r = resultFrom(x)
  if (needToAddTwo)
    r + 2
  else
    r + 1
}
```

And you know, as soon as the branches get too complex, just as was said in the
discussion on `return`, that's a sign that the *code smells* and is in need
of refactoring, which is a good thing.

### 2.4. SHOULD NOT define useless traits

There was this Java Best Practice that said "*program to an interface,
not to an implementation*", a best practice that has been cargo-cult-ed
to the point that people started defining completely useless
interfaces in their code. Generally speaking, that rule is healthy,
but it refers to the general engineering need of hiding implementation
details especially details of modifying state (encapsulation) and not
to slap interface declarations that often leak implementation details
anyway.

Defining traits is also a burden for readers of that code, because it
signals a need for polymorphism. Example:

```scala
trait PersonLike {
  def name: String
  def age: Int
}

case class Person(name: String, age: Int)
  extends PersonLike
```

Readers of this code might come to the conclusion that there are
instances in which overriding `PersonLike` is desirable. That couldn't
be further from the truth - `Person` is perfectly described by its
case class as a data-structure without behavior. In other words it
describes the shape of your data and if you need to override this
shape for some unknown reason, then this trait is badly defined
because it imposes the shape of your data and that's about the only
thing you can override. You can always come up with traits later, if
you're in need of polymorphism, after your needs evolve.

And if you're thinking that you may need to override the source of
this (as in to fetch the person's `name` from the DB on first access),
OMG don't do that!

Note that I'm not talking about algebraic data structures (i.e. sealed
traits that are signaling a closed set of choices - like `Option`).

Even in those cases in which you think the issue is clear-cut, it may
not be. Lets take this example:

```scala
trait DBService {
  def getAssets: Future[Seq[(AssetConfig, AssetPersistedState)]]

  def persistFlexValue(flex: FlexValue): Future[Unit]
}
```

This snippet is taken from real-world code - we've got a `DBService`
that handles either queries or persistence in a database. Those two
methods are actually unrelated, so if you only need to fetch the
assets, why depend on things you don't need in components that require
DB interaction?

Lately my code looks a lot like this:

```scala
final class AssetsObservable
    (f: => Future[Seq[(AssetConfig, AssetPersistedState)]])
  extends Observable[AssetConfigEvent] {

  // ...
}

object AssetsObservable {
  // constructor
  def apply(db: DBService) = new AssetsObservable(db.getAssets)
}
```

See, I do not need to mock an entire `DBService` in order to test the
above.

### 2.5. MUST NOT use "var" inside a case class

Case classes are syntactic sugar for defining classes in which - all
constructor arguments are public and immutable and thus part of the
value's identity, have structural equality, a corresponding hashCode
implementation and apply/unapply auto-generated functions provided by
the compiler.

By doing this:

```scala
case class Sample(str: String, var number: Int)
```

You just broke its equality and hashCode operation. Now try using it
as a key in a map.

As a general rule of thumb, structural equality only works for
immutable things, because the equality operation must be stable (and
not change according to the object's history). Case classes are for
strictly immutable things. If you need to mutate stuff, don't use case
classes.

In the approximate words of Fogus in "The Joy of Clojure" or Baker in
his paper from 1993: if any two mutable objects resolve as being equal
now, then there’s no guarantee that they will a moment from now. And
if two objects aren’t equal forever, then they’re technically never
equal ;-)

### 2.6. SHOULD NOT declare abstract "var" members

It's a bad practice to declare abstract vars in abstract classes or
traits. Do not do this:

```scala
trait Foo {
  var value: String
}
```

Instead, prefer to declare abstract things as `def`:

```scala
trait Foo {
  def value: String
}

// can then be overridden as anything
class Bar(val value: String) extends Foo
```

The reason has to do with the imposed restriction - a `var` can only
be overridden with a `var`. The way to allow freedom to choose on
inheritance is to use `def` for abstract members.  And why would you
impose the restriction to use a `var` on those that inherit from your
interface. `def` is generic so use it instead.

### 2.7. MUST NOT throw exceptions for validations of user input or flow control

Two reasons:

1. it goes against the principles of structured programming as a
   routine ends up having multiple exit points and are thus harder to
   reason about - with the stack unwinding happening being an awful and
   often unpredictable side-effect
2. exceptions aren't documented in the function's signature - Java
   tried fixing this with the checked exceptions concept, which in
   practice was awful as people simply ignored them

Exceptions are useful for only one thing - signaling unexpected errors
(bugs) up the stack, such that a supervisor can catch those errors and
decide to do things, like log the errors, send notifications,
restarting the guilty component, etc...

As an appeal to authority, it's reasonable to reference
[Functional Programming with Scala](http://www.manning.com/bjarnason/),
chapter 4.

### 2.8. MUST NOT catch Throwable when catching Exceptions

Never, never, never do this:

```scala
try {
 something()
} catch {
 case ex: Throwable =>
   blaBla()
}
```

Never catch `Throwable` because we could be talking about extremely
fatal exceptions that should never be caught and that should crash the
process. For example if the JVM throws an out of memory error, even if
you re-throw that exception in that catch clause, it may be too late -
given that the process is out of memory, the garbage collector
probably took over and froze everything, with the process ending in a
zombie unrecoverable state. Which means that an external supervisor
(like Upstart) will not get an opportunity to restart it.

Instead do this:

```scala
import scala.util.control.NonFatal

try {
 something()
} catch {
 case NonFatal(ex) =>
   blaBla()
}
```

### 2.9. MUST NOT use "null"

You must avoid using `null`. Prefer Scala's `Option[T]` instead. Null
values are error prone, because the compiler cannot protect
you. Nullable values that happen in function definitions are not
documented in those definitions. So avoid doing this:

```scala
def hello(name: String) =
  if (name != null)
    println(s"Hello, $name")
  else
    println("Hello, anonymous")
```

As a first step, you could be doing this:

```scala
def hello(name: Option[String]) = {
  val n = name.getOrElse("anonymous")
  println(s"Hello, $n")
}
```

The point of using `Option[T]` is that the compiler forces you to deal
with it, one way or another:

1. you either have to deal with it right away (e.g. by providing a
   default, throwing an exception, etc..)
2. or you can propagate the resulting `Option` up the call stack

Also remember that `Option` is just like a collection of 0 or 1
elements, so you can use foreach, which is totally idiomatic:

```scala
val name: Option[String] = ???

for (n <- name) {
  // executes only when the name is defined
  println(n)
}
```

Combining multiple options is also easy:

```scala
val name: Option[String] = ???
val age: Option[Int] = ???

for (n <- name; a <- age)
  println(s"Name: $n, age: $a")
```

And since `Option` is seen as an `Iterable` too, you can use `flatMap`
on collections to get rid of `None` values:

```scala
val list = Seq(1,2,3,4,5,6)

list.flatMap(x => Some(x).filter(_ % 2 == 0))
// => 2,4,6
```

### 2.10. MUST NOT use `Option.get`

You might be tempted to do this:

```scala
val someValue: Option[Double] = ???

// ....
val result = someValue.get + 1
```

Don't ever do that, since you're trading a `NullPointerException` for a
`NoSuchElementException` and that defeats the purpose of using
`Option` in the first place.

Alternatives:

1. using `Option.getOrElse`
2. using `Option.fold`
3. using pattern matching and dealing with the `None` branch explicitly
4. not taking the value out of its optional context

As an example for (4), not taking the value out of its context means
this:

```scala
val result = someValue.map(_ + 1)
```

### 2.11. MUST NOT use Java's Date or Calendar, instead use `java.time` (JSR-310)

Java's Date and Calendar classes from the standard library are awful
because:

1. resulting objects are mutable, which doesn't make sense for
   expressing a date, which should be a value (how would you feel if
   you had to work with StringBuffer everywhere you have Strings?)
2. months numbering is zero based
3. Date in particular does not keep timezone info, so Date values are completely useless
4. it doesn't make a difference between GMT and UTC
5. years are expressed as 2 digits instead of 4

Instead, always use the [`java.time`](https://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html) API
introduced in Java 8 - or if you're stuck in pre-Java 8 land, use [Joda-Time](http://www.joda.org/joda-time/), which is
its spiritual ancestor.

### 2.12. SHOULD NOT use Any or AnyRef or isInstanceOf / asInstanceOf

Avoid using Any or AnyRef or explicit casting, unless you've got a
really good reason for it. Scala is a language that derives value from
its expressive type system, usage of Any or of typecasting represents
a hole in this expressive type system and the compiler doesn't know
how to help you there. In general, something like this is bad:

```scala
val json: Any = ???

if (json.isInstanceOf[String])
  doSomethingWithString(json.asInstanceOf[String])
else if (json.isInstanceOf[Map])
  doSomethingWithMap(json.asInstanceOf[Map])
else
  ???
```

Often we are using Any when doing deserialization. Instead of working
with Any, think about the generic type you want and the set of
sub-types you need, and come up with an Algebraic Data-Type:

```scala
sealed trait JsValue

case class JsNumber(v: Double) extends JsValue
case class JsBool(v: Boolean) extends JsValue
case class JsString(v: String) extends JsValue
case class JsObject(map: Map[String, JsValue]) extends JsValue
case class JsArray(list: Seq[JsValue]) extends JsValue
case object JsNull extends JsValue
```

Now, instead of operating on Any, we can do pattern matching on
JsValue and the compiler can help us here on missing branches, since
the choice is finite. This will trigger a warning on missing branches:

```scala
val json: JsValue = ???
json match {
  case JsString(v) => doSomethingWithString(v)
  case JsNumber(v) => doSomethingWithNumber(v)
  // ...
}
```

### 2.13. MUST serialize dates as either Unix timestamp, or as ISO 8601

Unix timestamps, provided that we are talking about the number of
seconds or milliseconds since 1970-01-01 00:00:00 UTC (with emphasis
on UTC) are a decent cross-platform serialization format. It does have
the disadvantage that it has limits in what it can express. ISO-8601
is a decent serialization format supported by most libraries.

Avoid anything else and also when storing dates without a timezone
attached (like in MySQL), always express that info in UTC.

### 2.14. MUST NOT use magic values

Although not uncommon in other languages to use "magic" (special)
values like `-1` to signal particular outcomes, in Scala there are a
range of types to make intent clear. `Option`, `Either`, `Try` are
examples. Also, in case you want to express more than a boolean
success or failure, you can always come up with an algebraic data
type.

Don't do this:

```scala
val index = list.find(someTest).getOrElse(-1)
```

### 2.15. SHOULD NOT use "var" as shared state

Avoid using "var" at least when speaking about shared mutable
state. Because if you do have shared state expressed as vars, you'd
better synchronize it and it gets ugly fast. Much better is to avoid
it. In case you really need mutable shared state, use an atomic
reference and store immutable things in it. Also checkout
[Scala-STM](https://nbronson.github.io/scala-stm/).

So instead of something like this:

```scala
class Something {
  private var cache = Map.empty[String, String]
}
```

If you can't really avoid that variable, prefer doing this:

```scala
import java.util.concurrent.atomic._

class Something {
  private val cache =
    new AtomicReference(Map.empty[String, String])
}
```

Yes, it introduces overhead due to the synchronization required, which
in the case of an atomic reference means spin loops. But it will save
you from lots and lots of headaches later. And it's best to avoid
mutation entirely.

### 2.16. Public functions SHOULD have an explicit return type

Prefer this:

```scala
def someFunction(param1: T1, param2: T2): Result = {
  ???
}
```

To this:

```scala
def someFunction(param1: T1, param2: T2) = {
  ???
}
```

Yeah, type inference on the result of a function is great and all, but
for public methods:

1. it's not OK to rely on an IDE or to inspect the implementation in
   order to see the returned type
2. Scala currently infers the most specialized type possible, because
   in Scala the return type on functions is covariant, so you might
   actually get a really ugly type back

For example, what is the returned type of this function:

```scala
def sayHelloRunnable(name: String) = new Runnable {
  def sayIt() = println(s"Hello, $name")
  def run() = sayIt()
}
```

Do you think it's `Runnable`?
Wrong, it's `Runnable{def sayIt(): Unit}`.

As a side-effect, this also increases compilation times, as whenever
`sayHelloRunnable` changes implementation, it also changes the
signature so everything that depends on it must be recompiled.

### 2.17. SHOULD NOT define case classes nested in other classes

It is tempting, but you should almost never define nested case classes
inside another object/class because it messes with Java's
serialization. The reason is that when you serialize a case class it
closes over the "this" pointer and serializes the whole object, which
if you are putting in your App object means for every instance of a
case class you serialize the whole world.

And the thing with case classes specifically is that:

1. one expects a case class to be immutable (a value, a fact) and hence
2. one expects a case class to be easily serializable

Prefer flat hierachies.

### 2.18 MUST NOT include classes, traits and objects inside package objects

Classes, including case classes, traits and objects do not belong
inside package objects. It is unnecessary, confuses the compiler and
is therefore discouraged. For example, refrain from doing the
following:

```scala
package foo

package object bar {
  case object FooBar
}
```

The same effect is achieved if all artifacts are inside a plain package:

```scala
package foo.bar

case object FooBar
```

Package objects should only contain value, method and type alias
definitions, etc.  Scala allows multiple public classes in a single
file, and the convention is to have the first letter of the filename
be lowercase in such cases.

#### Implicit value classes can be defined in a package object

In one rare circumstance, it makes sense to include classes defined directly in a `package object`. The reason for that is that implicit classes need to be nested inside another object/class, you cannot define a top level implicit in Scala. Nesting implicit value classes inside a `package object` also allows us to create a great importing experience for a library, as the single import of the package object will bring in all the necessary implicits.

There is also no way around this, as defining the implicit value class means we cannot effectively define the implicit and the class separately, the whole point is to let the compiler avoid the runtime boxing by generating all the "right code" at compile time, by predicting the runtime boxing. This is the optimal way to achieve a pimp my library pattern performance wise, and we need the entire syntax "in one place".

```scala

package object dsl {
  implicit class DateTimeAugmenter(val date: Datetime) extends AnyVal {
    def yesterday: DateTime = date.plusDays(-1)
  }
}

```

### 2.19 SHOULD use head/tail and init/last decomposition only if they can be done in constant time and memory

Example of head/tail decomposition:

```scala
def recursiveSumList(numbers: List[Int], accumulator: Int): Int =
  numbers match {
    case Nil =>
      accumulator

    case head :: tail =>
      recursiveSumList(tail, accumulator + head)
  }
```

`List` has a special head/tail extractor `::` because `List`s are **made** by always appending an element to the front of the list:

```scala
val numbers = 1 :: 2 :: 3 :: Nil
```

This is the same as:

```scala
val numbers = Nil.::(3).::(2).::(1)
```

For this reason, both `head` and `tail` on a list need only constant time and memory! These operations are `O(1)`.

There is another head/tail extractor called `+:` that works on any `Seq`:

```scala
def recursiveSumSeq(numbers: Seq[Int], accumulator: Int): Int =
  numbers match {
    case Nil =>
      accumulator

    case head +: tail =>
      recursiveSumSeq(tail, accumulator + head)
  }
```

You can find the implementation of `+:` [here](https://github.com/scala/scala/blob/v2.12.4/src/library/scala/collection/SeqExtractors.scala). The problem is that other collections than `List` do not necessarily head/tail-decompose in constant time and memory, e.g. an `Array`:

```scala
val numbers = Array.range(0, 10000000)

recursiveSumSeq(numbers, 0)
```

This is highly inefficient: each `tail` on an `Array` takes `O(n)` time and memory, because every time a new array needs to be created!

Unfortunately, the Scala collections library permits these kinds of inefficient operations. We have to keep an eye out for them.

---

An example for an efficient init/last decomposition is `scala.collection.immutable.Queue`. It is backed by two `List`s and the efficiency of `head`, `tail`, `init` and `last` is *amortized constant* time and memory, as explained in the [Scala collection performance characteristics](http://docs.scala-lang.org/overviews/collections/performance-characteristics.html).

I don't think that init/last decomposition is all that common. In general, it is analogue to head/tail decomposition. The init/last deconstructor for any `Seq` is `:+`.


### 2.20 MUST NOT use `Seq.head`

You might be tempted to this:

```scala
val userList: List[User] = ???

// ....
val firstName = userList.head.firstName
```

Don't ever do this, as this will throw `NoSuchElementException` if the sequence is empty.

Alternatives:

1. using `Seq.headOption` possibly combined with `getOrElse` or pattern matching

    Example:
    
    ```scala
    val firstName = userList.headOption match {
        case Some(user) => user.firstName
        case _ => "Unknown"
      }
    ```

2. using pattern matching with the cons operator `::` if you're dealing with a `List`

    Example:
    
    ```scala
    val firstName = userList match {
     case head :: _ => head.firstName
     case _ => "Unknown"
    }
    ```

3. using `NonEmptyList` if it is required that the list should never be empty. (See [cats](https://typelevel.org/cats/datatypes/nel.html), [scalaz](https://github.com/scalaz/scalaz/blob/series/7.3.x/core/src/main/scala/scalaz/NonEmptyList.scala), ...)

### 2.21 Case classes SHOULD be final

Extending a case class will lead to unexpected behaviour. Observe the following:
```scala
scala> case class Foo(v:Int)
defined class Foo

scala> class Bar(v: Int, val x: Int) extends Foo(v)
defined class Bar

scala> new Bar(1, 1) == new Bar(1, 1)
res25: Boolean = true

scala> new Bar(1, 1) == new Bar(1, 2)
res26: Boolean = true
// ????
scala> new Bar(1,1) == Foo(1)
res27: Boolean = true

scala> class Baz(v: Int) extends Foo(v)
defined class Baz

scala> new Baz(1) == new Bar(1,1)
res29: Boolean = true //???

scala> println (new Bar(1,1))
Foo(1) // ???

scala> new Bar(1,2).copy()
res49: Foo = Foo(1) // ???
```
Credits:[Why case classes should be final](https://stackoverflow.com/a/34562046/3856808)

So by default case classes should always be defined as final. 

Example:

```scala
final case class User(name: String, id: Long)
```

### 2.22 SHOULD NOT use `scala.App`

`scala.App` is often used to denote the entrypoint of the application:

```scala
object HelloWorldApp extends App {
  println("hello, world!")
}
```

`DelayedInit` one of the mechanisms used to implement `scala.App` [has been deprecated](https://github.com/scala/scala/pull/3563).
Any variables defined in the object body will be available as fields, unless the `private` access modifier is applied.
Prefer the simpler alternative of defining a main method:

```scala
object HelloWorldApp {
  def main(args: Array[String]): Unit = println("hello, world!")
}
```
