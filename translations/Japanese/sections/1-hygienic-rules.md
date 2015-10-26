## 1. Hygienic Rules

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

These are general purpose hygienic rules that transcend the language
or platform rules. Programming language is a form of communication,
targeting computer systems, but also your colleagues and your future
self, so respecting these rules is just like washing your hands after
going to the bathroom.

### 1.1. SHOULD enforce a reasonable line length

There's a whole science on typography which says that people lose
their focus when the line of text is too wide, a long line makes it
difficult to gauge where the line starts or ends and it makes it
difficult to continue on the next line below it, as your eyes have to
move a lot from the right to the left. It also makes it difficult to
scan for important details.

In typography, the optimal line length is considered to be somewhere
between 50 and 70 chars.

In programming, we've got indentation, so it's not feasible to impose
a 60 chars length for lines. 80 chars is usually acceptable, but not
in Scala, because in Scala we use a lot of closures and if you want
long and descriptive names for functions and classes, well, 80 chars
is way too short.

120 chars, as IntelliJ IDEA is configured by default, may be too wide
on the other hand.  Yes, I know that we've got 16:9 wide monitors, but
this doesn't help readability and with shorter lines we can put these
wide monitors to good use when doing side-by-side diffs. And with long
lines it takes effort to notice important details that happen at the
end of those lines.

So as a balance:

- strive for 80 chars as the soft limit and if it gets ugly,
- then 100 chars is enough, except for ...
- function signatures, which can get really ugly if limited

On the other hand, anything that goes beyond 120 chars is an abomination.

### 1.2. MUST NOT rely on a SBT or IDE plugin to do the formatting for you

IDEs and SBT plugins can be of great help, however if you're thinking
about using one to automatically format your code, beware.

You won’t find a plugin that is able to infer the developer’s intent,
since that requires a human-like understanding of the code and would be 
near impossible to make. The purpose of proper indentation and formatting 
isn't to follow some rigid rules set upon you in a cargo-cult way, but to
make the code more logical, more readable, more
approachable. Indentation is actually an art form, which is not awful
since all you need is a nose for awful code and the urge of fixing
it. And it is in the developer's job description to make sure that his
code doesn't stink.

So automated means are fine, BUT BE CAREFUL to not ruin other people's
carefully formatted code, otherwise I'll slap you in prose.

Lets think about what I said - if the line is too long, how is a
plugin supposed to break it? Lets talk about this line (real code):

```scala
    val dp = new DispatchPlan(new Set(filteredAssets), start = startDate, end = endDate, product, scheduleMap, availabilityMap, Set(activationIntervals.get), contractRepository, priceRepository)
```

In most cases, a plugin will just do truncation and I've seen a lot of these in practice:

```scala
    val dp = new DispatchPlan(Set(filteredAssets), start =
      startDate, end = endDate, product, scheduleMap, availabilityMap,
      Set(activationIntervals), contractRepository, priceRepository)
```

Now that's not readable, is it? I mean, seriously, that looks like
barf. And that's exactly the kind of output I see coming from people
relying on plugins to work. We could also have this version:

```scala
    val dp = new DispatchPlan(
      Set(filteredAssets),
      startDate,
      endDate,
      product,
      scheduleMap,
      availabilityMap,
      Set(activationIntervals),
      contractRepository,
      priceRepository
    )
```

Looks much better. But truth is, this isn't so good in other
instances. Like say we've got a line that we want to break:

```scala
   val result = service.something(param1, param2, param3, param4).map(transform)
```

Now placing those parameters on their own line is awful, no matter how you deal with it:

```scala
    // awful because that transform call is not visible
    val result = service.something(
      param1,
      param2,
      param3,
      param4).map(transform)

    // awful because it breaks the logical flow
    val result = service.something(
      param1,
      param2,
      param3,
      param4
    ).map(transform)
```

This would be much better:

```scala
    val result = service
      .something(param1, param2, param3, param4)
      .map(transform)
```

Now that's better, isn't it? Of course, sometimes that call is so long
that this doesn't cut it. So you need to resort to a temporary value
of some sort, e.g...

```scala
    val result = {
      val instance =
        object.something(
          myAwesomeParam1,
          otherParam2,
          someSeriousParam3,
          anEvenMoreSoParam4,
          lonelyParam5,
          catchSomeFn6,
          startDate7
        )

      for (x <- instance) yield
        transform(x)
    }
```

Of course, sometimes if the code stinks so badly, you need to get into
refactoring - as in, maybe too many parameters are too much for a
function ;-)

And we are talking strictly about line lengths - once we get into
other issues, things get even more complicated. So really, you won't
find a plugin that does this analysis and that can make the right
decision for you.

### 1.3. SHOULD break long functions

Ideally functions should only be a couple of lines long. If the lines
get too big, then we need to break them into smaller functions and
give them a name.

Note that in Scala we don't necessarily have to make such intermediate
functions available in other scopes, the purpose here is to primarily
aid readability, so in Scala we can do inner-functions to break logic
into pieces.

### 1.4. MUST NOT introduce spelling errors in names and comments

Spelling errors are freakishly annoying, interrupting a reader's flow.
Use a spell-checker. Intelligent IDEs have built-in
spell-checkers. Note the underlined spelling warnings and fix them.

### 1.5. Names MUST be meaningful

*"There are only two hard things in Computer Science: cache
invalidation and naming things."* -- Phil Karlton

We've got three guidelines here:

1. give descriptive names, but don't go overboard, four words is a
   little too much already
2. you can be terse in naming if the type / purpose can be easily
   inferred from the immediate context, or if there's already an
   established convention
3. if going the descriptive route, don't do bullshit words that are
   meaningless

For example this is acceptable:

```scala
for (p <- people) yield
  transformed(p)
```

We can see that `p` is a person from the immediate context, so a short
one letter name is OK. This is also acceptable because `i` is an
established convention to use as an index:

```scala
for (i <- 0 until limit) yield ???
```

This is in general not acceptable, because usually with tuples the
naming of the collection doesn't reflect well what's contained (if you
haven't given those elements a name, then as a consequence the
collection itself is going to have a bad name):

```
someCollection.map(_._2)
```

Implicit parameters on the other hand are OK with short names, because
being passed implicitly, we don't care about them unless they are
missing:

```scala
def query(id: Long)(implicit ec: ExecutionContext, c: WSClient): Future[Response]
```

This is not acceptable because the name is utterly meaningless, even
if there's a clear attempt at being descriptive:

```scala
def processItems(people: Seq[Person]) = ???
```

It's not acceptable because the naming of this function indicates a
side-effect (`process` is a verb indicating a command), yet it doesn't
describe what we are doing with those `people`. The `Items` suffix is
meaningless, because we might have said `processThingy`,
`processRows`, `processStuff` and it would still say exactly the same
thing - absolutely nothing. It also increases visual clutter, as more
words is more text to read and meaningless words are just noise.

Properly chosen descriptive names - good. Bullshit names - bad.
