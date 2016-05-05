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

### 2.11. MUST NOT use Java's Date or Calendar, instead use Joda-Time or JSR-310

Java's Date and Calendar classes from the standard library are awful
because:

1. resulting objects are mutable, which doesn't make sense for
   expressing a date, which should be a value (how would you feel if
   you had to work with StringBuffer everywhere you have Strings?)
2. months numbering is zero based
3. Date in particular does not keep timezone info, so Date values are completely useless
4. it doesn't make a difference between GMT and UTC
5. years are expressed as 2 digits instead of 4

Always use [Joda-Time](http://www.joda.org/joda-time/) - of if you can
afford to switch to Java 8, there's a shinny new
[JSR-310](http://www.threeten.org/) that's based on Joda-Time and that
will be the new standard once people adopt Java 8.

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
