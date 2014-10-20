<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" />

## 2. Application Architecture

### 2.1. SHOULD NOT use the Cake Pattern

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
attention to lifecycle issues. What happens in practice is sloppiness,
with the result being a big hairball. It's awesome that Scala allows
you to do things like the Cake pattern, highlighting the real power of
OOP, but just because you can doesn't mean you should, because if the
purpose is doing dependency injection and decoupling between various
components, you'll fail hard and impose that maintainance burden on
your colleagues.

In fact, one should strive to not do dependency injection at all, or
to do it only at the edges (like Play's controllers). Because if a
component depends on too many things, that's *code smell*. If a
component depends on hard to initialize arguments, that's also *code
smell*. Don't hide painful things under the rug, fix it instead.

### 2.2. MUST NOT put things in Play's Global

I'm seeing this over and over again.

Folks,
[Play's Global](https://www.playframework.com/documentation/2.3.x/ScalaGlobal)
object is not a bucket in which you can shove your orphaned pieces of
code. Its purpose is to hook into Play's configuration and lifecycle,
nothing more.

Come up with your own freaking namespace for your utilities.
