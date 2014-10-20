# Scala Best Practices

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="100" height="100" />

A collection of best practices, friendly to people that want to
contribute.

- Version: `0.2`
- Updated at: `2014-10-21`

## Table of Contents

- [0. Preface](0-preface.md)
  - [0.1 MUST NOT follow advice blindly](0-preface.md#01-must-not-follow-advice-blindly)

- [1. Hygienic Rules](1-hygienic-rules.md)
  - [1.1. SHOULD enforce a reasonable line length](1-hygienic-rules.md#11-should-enforce-a-reasonable-line-length)
  - [1.2. MUST NOT rely on a SBT or IDE plugin to do the formatting for you](1-hygienic-rules.md#12-must-not-rely-on-a-sbt-or-ide-plugin-to-do-the-formatting-for-you)
  - [1.3. SHOULD break long functions](1-hygienic-rules.md#13-should-break-long-functions)
  - [1.4. MUST NOT introduce spelling errors in names and comments](1-hygienic-rules.md#14-must-not-introduce-spelling-errors-in-names-and-comments)
  - [1.5. Names MUST be meaningful](1-hygienic-rules.md#15-names-must-be-meaningful)
  
- [2. Language Rules](2-language-rules.md)
  - [2.1. MUST NOT use "return"](2-language-rules.md#21-must-not-use-return)
  - [2.2. SHOULD use immutable data structures](2-language-rules.md#22-should-use-immutable-data-structures)
  - [2.3. SHOULD NOT update a "var" using loops or conditions](2-language-rules.md#23-should-not-update-a-var-using-loops-or-conditions)
  - [2.4. SHOULD NOT define useless traits](2-language-rules.md#24-should-not-define-useless-traits)
  - [2.5. MUST NOT use "var" inside a case class](2-language-rules.md#25-must-not-use-var-inside-a-case-class)
  - [2.6. SHOULD NOT declare abstract val or var or lazy val members](2-language-rules.md#26-should-not-declare-abstract-val-or-var-or-lazy-val-members)
  - [2.7. MUST NOT throw exceptions for validations of user input or flow control](2-language-rules.md#27-must-not-throw-exceptions-for-validations-of-user-input-or-flow-control)
  - [2.8. MUST NOT catch Throwable](2-language-rules.md#28-must-not-catch-throwable-when-catching-exceptions)
  - [2.9. MUST NOT use "null"](2-language-rules.md#29-must-not-use-null)
  - [2.10. MUST NOT use Java's Date or Calendar, instead use Joda-Time](2-language-rules.md#210-must-not-use-javas-date-or-calendar-instead-use-joda-time)
  - [2.11. SHOULD NOT use Any or AnyRef or isInstanceOf / asInstanceOf](2-language-rules.md#211-should-not-use-any-or-anyref-or-isinstanceof--asinstanceof)
  - [2.12. MUST serialize dates as either Unix Timestamp or ISO 8601](2-language-rules.md#212-must-serialize-dates-as-either-unix-timestamp-or-as-iso-8601)
  - [2.13. MUST NOT use magic values](2-language-rules.md#213-must-not-use-magic-values)
  - [2.14. SHOULD NOT use "var" as shared state](2-language-rules.md#214-should-not-use-var-as-shared-state)
  - [2.15. MUST be mindful of the garbage collector](2-language-rules.md#215-must-be-mindful-of-the-garbage-collector)

- [3. Application Architecture](3-architecture.md)
  - [3.1. SHOULD NOT use the Cake pattern](3-architecture.md#31-should-not-use-the-cake-pattern)
  - [3.2. MUST NOT put things in Play's Global](3-architecture.md#32-must-not-put-things-in-plays-global)

- [4. Concurrency and Parallelism](4-concurrency-parallelism.md)
  - [4.1. SHOULD avoid concurrency like the plague it is](4-concurrency-parallelism.md#41-should-avoid-concurrency-like-the-plague-it-is)
  - [4.2. SHOULD use appropriate abstractions only where suitable](4-concurrency-parallelism.md#42-should-use-appropriate-abstractions-only-where-suitable---future-actors-rx)
  - [4.3. MUST NOT wrap purely CPU-bound operations in Futures](4-concurrency-parallelism.md#43-must-not-wrap-purely-cpu-bound-operations-in-futures)
  - [4.4. MUST use Scala's BlockContext on blocking I/O](4-concurrency-parallelism.md#44-must-use-scalas-blockcontext-on-blocking-io)
  - [4.5. SHOULD NOT block](4-concurrency-parallelism.md#45-should-not-block)
  - [4.6. All public APIs SHOULD BE thread-safe](4-concurrency-parallelism.md#46-all-public-apis-should-be-thread-safe)
  - [4.7. SHOULD avoid contention on shared reads](4-concurrency-parallelism.md#47-should-avoid-contention-on-shared-reads)
  - [4.8. MUST provide a clearly defined and documented protocol for each component or actor that communicates over async boundaries](4-concurrency-parallelism.md#48-must-provide-a-clearly-defined-and-documented-protocol-for-each-component-or-actor-that-communicates-over-async-boundaries)
  - [4.9. SHOULD always prefer single producer scenarios](4-concurrency-parallelism.md#49-should-always-prefer-single-producer-scenarios)
  - [4.10. MUST NOT hard-code the thread-pool / execution context](4-concurrency-parallelism.md#410-must-not-hardcode-the-thread-pool--execution-context)

- [5. Akka Actors](5-actors.md)
  - [5.1. MUST evolve the state of actors only in response to messages received from the outside](5-actors.md#51-must-evolve-the-state-of-actors-only-in-response-to-messages-received-from-the-outside)
  - [5.2. SHOULD mutate state in actors only with "context.become"](5-actors.md#52-should-mutate-state-in-actors-only-with-contextbecome)
  - [5.3. MUST NOT leak the internal state of an actor in asynchronous closures](5-actors.md#53-must-not-leak-the-internal-state-of-an-actor-in-asynchronous-closures)
  - [5.4. SHOULD do back-pressure](5-actors.md#54-should-do-back-pressure)

---

## Contribute

Open an issue to make suggestions, or create a pull request ;-)

---

Copyright &copy; 2014, Some Rights Reserved.<br />Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
