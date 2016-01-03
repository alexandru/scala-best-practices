## 3. Application Architecture

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 3.1. SHOULD NOT use the Cake Pattern

The Cake Pattern is a very
[good idea in theory](https://www.youtube.com/watch?v=yLbdw06tKPQ) -
using traits as modules that can be composed, giving you the ability
to override `import`, with compile-time dependency injection as a
side-effect.

In practice all the Cake implementations I've seen have been awful,
new projects should steer away and existing projects should be
migrated off Cake.

People are not implementing Cake correctly, being a poorly understood
design pattern. I haven't seen Cake implementations in which the
traits are designed to be abstract modules, or that pay proper
attention to life-cycle issues. What happens in practice is sloppiness,
with the result being a big hairball. It's awesome that Scala allows
you to do things like the Cake pattern, highlighting the real power of
OOP, but just because you can doesn't mean you should, because if the
purpose is doing dependency injection and decoupling between various
components, you'll fail hard and impose that maintenance burden on
your colleagues.

For example, this is a common occurrence in Cake:

```scala
trait SomeServiceComponent {
  type SomeService <: SomeServiceLike
  val someService: SomeService // abstract

  trait SomeServiceLike {
    def query: Rows
  }
}

trait SomeServiceComponentImpl extends SomeServiceComponent {
  self: DBServiceComponent =>

  val someService = new SomeService

  class SomeService extends SomeServiceLike {
    def query = dbService.query
  }
}
```

In the example above `someService` is effectively a
[singleton](https://en.wikipedia.org/wiki/Singleton_pattern) and a genuine
one, because it's probably missing *life-cycle management*. And if by reading
this code your alarms weren't set off by a singleton missing life-cycle
management, well, be acquainted with the ugly secret of most Cake
implementations. And for those conscious few that are doing this correctly,
they end up in JVM initialization hell.

But that's not the only problem. The bigger problem is that developers are
lazy, so you end up with huge components having dozens of dependencies
and responsibilities, because Cake encourages this. And after the original
developers that did this damage move from the project, you end up with
other, smaller components, that duplicate the functionality of the original
components, just because the original components are hell-like to test because
you have to mock or stub too many things (another code smell). And you've got
this forever repeating cycle, with developers ending up hating the code base,
doing the minimal amount of work required to accomplish their tasks, ending
up with other big, ugly and fundamentally flawed components. And because of
the tight coupling that Cake naturally induces, they won't be easy to refactor.

So why do the above when something like this is much more readable
and common sense:

```scala
class SomeService(dbService: DBService) {
  def query = dbService.query
}
```

Or if you really need abstract stuff (but please read
[rule 2.4](2-language-rules.md#24-should-not-define-useless-traits)
on not defining useless traits):

```scala
trait SomeService {
  def query: Rows
}

object SomeService {
  /** Builder for [[SomeService]] */
  def apply(dbService: DBService): SomeService =
    new SomeServiceImpl

  private final class SomeServiceImpl(dbService: DBService)
    extends SomeService {
    def query: Rows = dbService.query
  }
}
```

Are your dependencies going crazy? Are those constructors starting
to hurt? That's a feature. It is called "*pain driven development*"
(PDD for short :-)). It's a sign that the architecture is not OK
and the various dependency injection libraries or the Cake pattern
are not fixing the problem, but the symptoms, by hiding the junk under
the rug.

So prefer plain old and reliable *constructor arguments*. And if you do
need to use dependency injection libraries, then do it at the edges
(like in Play's controllers). Because if a component depends on too many
things, that's *code smell*. If a component depends on hard to initialize
arguments, that's *code smell*. If you need to mock or stub interfaces in your
tests just to test the pure business logic, that's probably *code smell* ;-)

Don't hide painful things under the rug, fix it instead.

### 3.2. MUST NOT put things in Play's Global

I'm seeing this over and over again.

Folks,
[Play's Global](https://www.playframework.com/documentation/2.3.x/ScalaGlobal)
object is not a bucket in which you can shove your orphaned pieces of
code. Its purpose is to hook into Play's configuration and life-cycle,
nothing more.

Come up with your own freaking namespace for your utilities.

### 3.3. SHOULD NOT apply optimizations without profiling

Profiling is a prerequisite for doing optimizations. Never work on
optimizations, unless through profiling you discover the actual
bottlenecks.

This is because our intuition about how the system behaves often fails
us and multiple effects could happen by applying optimizations without
having hard numbers:

- you could complicate the code or the architecture, thus making it
  harder to apply later optimizations globally
- your work could be in vain or it could actually lead to more
  performance degradation

Multiple strategies available and you should preferably do all of
them:

- a good profiler can tell you about bottlenecks that aren't obvious,
  my favorite being YourKit Profiler, but Oracle's VisualVM is free
  and often good enough
- collect metrics from the running production systems, by means of a
  library such as
  [Dropwizard Metrics](https://dropwizard.github.io/metrics/3.1.0/)
  and push them in something like
  [Graphite](http://graphite.wikidot.com/), a strategy that can lead
  you in the right direction
- compare solutions by writing benchmarking code, but note that
  benchmarking is not easy and you should at least use a library like
  [Google Caliper](https://code.google.com/p/caliper/)

Overall - measure, don't guess.

### 3.4. SHOULD be mindful of the garbage collector

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
 .filter(Set(a,b,c).contains)
 .map(_.name)
```

First of all, this creates a Set every single time, on each element of
our collection. Second of all, filter and map can be compressed in one
operation, otherwise we end up with more garbage and more time spent
building the final collection:

```scala
val isIDValid = Set(a,b,c)

someCollection.collect {
  case x if isIDValid(x) => x.name
}
```

A generic example that often pops up, exemplifying useless traversals
and operators that could be compressed:

```scala
collection
  .filter(bySomething)
  .map(toSomethingElse)
  .filter(again)
  .headOption
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

BTW, there is an obvious solution for keeping expressiveness while
doing filtering and mapping - lazy collections, which in Scala means
[Stream](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Stream)
if you need memoization or
[Iterable](http://docs.oracle.com/javase/7/docs/api/java/lang/Iterable.html)
if you don't need memoization.

Also, make sure to read the
[Rule 3.3](#33-should-not-apply-optimizations-without-profiling) on
profiling.

### 3.5. MUST NOT use parameterless ConfigFactory.load() or access a Config object directly

It may be very tempting to call the oh-so-available-and-parameterless
`ConfigFactory.load()` method whenever you need to pull something from
the configuration, but doing so will boomerang back at you, for
instance when writing tests.

If you have
[`ConfigFactory.load()`](https://typesafehub.github.io/config/latest/api/com/typesafe/config/ConfigFactory.html#load--)
scattered all around your classes they are basically loading the
default configuration when your code runs, which more often than not,
is not what you really want to happen in a testing environment, where
you need to have a modified configuration loaded (e.g., different
timeouts, different implementations, different IPs, etc.).

NEVER do this:

```scala
class MyComponent {
  private val ip = ConfigFactory.load().getString("myComponent.ip")
}
```

One way to go about dealing with it, is to pass the `Config` instance
itself to whomever needs it, or have the needed values from it passed
in. The situation described here is in fact a flavor of the
[prefer dependency injection (DI) over Service Locator](http://stackoverflow.com/questions/1638919/how-to-explain-dependency-injection-to-a-5-year-old/1638961#1638961)
practice.

You can call `ConfigFactory.load()`, but from your application's root,
say in your `main()` (or equivalent) so that you don't have to hardcode your
configuration's filename.

Another good practice is to have domain specific config classes,
which are parsed from the general purpose, map-like, Config
objects. The benefit of this approach is that specialized config
classes faithfully represent your specific configuration needs, and
once parsed, allow you to work against compiled classes in a more
type-safe way (where "safer" means you do `config.ip`, instead of
`config.getString("ip")`).

This also has the benefit of clarity, as your domain specific config
class conveys the needed properties in a more explicit and readable
manner.

Consider the following example:

```scala
/** This is your domain specific config class, with a pre-defined set of
  * properties you've modeled according to your domain, as opposed to
  * a map-like properties bag
  */
case class AppConfig(
  myComponent: MyComponentConfig,
  httpClient: HttpClientConfig
)

/** Configuration for [[MyComponent]] */
case class MyComponentConfig(ip: String)

/** Configuration for [[HttpClient]] */
case class HttpClientConfig(
  requestTimeout: FiniteDuration,
  maxConnectionsPerHost: Int
)

object AppConfig {
  /** Loads your config.
    * To be used from `main()` or equivalent.
    */
  def loadFromEnvironment(): AppConfig =
    load(ConfigUtil.loadFromEnvironment())

  /** Load from a given Typesafe Config object */
  def load(config: Config): AppConfig =
    AppConfig(
        myComponent = MyComponentConfig(
          ip = config.getString("myComponent.ip")
        ),
        httpClient = HttpClientConfig(
          requestTimeout = config.getDuration("httpClient.requestTimeout", TimeUnit.MILLISECONDS).millis,
          maxConnectionsPerHost = config.getInt("httpClient.maxConnectionsPerHost")
        )
    )
}

object ConfigUtil {
  /** Utility to replace direct usage of ConfigFactory.load() */
  def loadFromEnvironment(): Config = {
    Option(System.getProperty("config.file"))
      .map(f => ConfigFactory.parseFile(f).resolve())
      .getOrElse(
        ConfigFactory.load(System.getProperty(
          "config.resource", "application.conf")))
  }
}

/** One component */
class HttpClient(config: HttpClientConfig) {
  ???
}

/** Another component, depending on your domain specific config.
  * Also notice the sane dependency injection ;-)
  */
class MyComponent(config: MyComponentConfig, httpClient: HttpClient) {
  ???
}
```

Benefits of this approach:

- the config objects are just immutable case classes of primitives that
  can be easily instantiated
- your components end up depending on concrete and type-safe configuration
  definitions related only to them, instead of receiving a monolithic and
  unsafe `Config` that contains everything and that's expensive to instantiate
- and now your IDE can help with documentation and discoverability
- and your compiler can help with spelling errors

NOTE about style: these configuration case classes tend to get big and to contain
primitives (e.g. ints, strings, etc.), so usage of named
parameters makes the code more resistant to change and less error-prone,
versus relying on positioning. The style of indentation chosen here makes
the instantiation look like a `Map` or a JSON object if you want.
