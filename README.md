# Scala Best Practices

[![Join the chat at https://gitter.im/alexandru/scala-best-practices](https://badges.gitter.im/alexandru/scala-best-practices.svg)](https://gitter.im/alexandru/scala-best-practices?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="100" height="100" />

A collection of best practices, friendly to people that want to
contribute.

- Version: `1.2`
- Updated at: `2016-06-08`

## Table of Contents

- [0. Preface](sections/0-preface.md)
  - [0.1 MUST NOT follow advice blindly](sections/0-preface.md#01-must-not-follow-advice-blindly)

- [1. Hygienic Rules](sections/1-hygienic-rules.md)
  - [1.1. SHOULD enforce a reasonable line length](sections/1-hygienic-rules.md#11-should-enforce-a-reasonable-line-length)
  - [1.2. MUST NOT rely on a SBT or IDE plugin to do the formatting for you](sections/1-hygienic-rules.md#12-must-not-rely-on-a-sbt-or-ide-plugin-to-do-the-formatting-for-you)
  - [1.3. SHOULD break long functions](sections/1-hygienic-rules.md#13-should-break-long-functions)
  - [1.4. MUST NOT introduce spelling errors in names and comments](sections/1-hygienic-rules.md#14-must-not-introduce-spelling-errors-in-names-and-comments)
  - [1.5. Names MUST be meaningful](sections/1-hygienic-rules.md#15-names-must-be-meaningful)

- [2. Language Rules](sections/2-language-rules.md)
  - [2.1. MUST NOT use "return"](sections/2-language-rules.md#21-must-not-use-return)
  - [2.2. SHOULD use immutable data structures](sections/2-language-rules.md#22-should-use-immutable-data-structures)
  - [2.3. SHOULD NOT update a "var" using loops or conditions](sections/2-language-rules.md#23-should-not-update-a-var-using-loops-or-conditions)
  - [2.4. SHOULD NOT define useless traits](sections/2-language-rules.md#24-should-not-define-useless-traits)
  - [2.5. MUST NOT use "var" inside a case class](sections/2-language-rules.md#25-must-not-use-var-inside-a-case-class)
  - [2.6. SHOULD NOT declare abstract "var" members](sections/2-language-rules.md#26-should-not-declare-abstract-var-members)
  - [2.7. MUST NOT throw exceptions for validations of user input or flow control](sections/2-language-rules.md#27-must-not-throw-exceptions-for-validations-of-user-input-or-flow-control)
  - [2.8. MUST NOT catch Throwable](sections/2-language-rules.md#28-must-not-catch-throwable-when-catching-exceptions)
  - [2.9. MUST NOT use "null"](sections/2-language-rules.md#29-must-not-use-null)
  - [2.10. MUST NOT use "Option.get"](sections/2-language-rules.md#210-must-not-use-optionget)
  - [2.11. MUST NOT use Java's Date or Calendar, instead use `java.time` (JSR-310)](sections/2-language-rules.md#211-must-not-use-javas-date-or-calendar-instead-use-javatime-jsr-310)
  - [2.12. SHOULD NOT use Any or AnyRef or isInstanceOf / asInstanceOf](sections/2-language-rules.md#212-should-not-use-any-or-anyref-or-isinstanceof--asinstanceof)
  - [2.13. MUST serialize dates as either Unix Timestamp or ISO 8601](sections/2-language-rules.md#213-must-serialize-dates-as-either-unix-timestamp-or-as-iso-8601)
  - [2.14. MUST NOT use magic values](sections/2-language-rules.md#214-must-not-use-magic-values)
  - [2.15. SHOULD NOT use "var" as shared state](sections/2-language-rules.md#215-should-not-use-var-as-shared-state)
  - [2.16. Public functions SHOULD have an explicit return type](sections/2-language-rules.md#216-public-functions-should-have-an-explicit-return-type)
  - [2.17. SHOULD NOT define case classes nested in other classes](sections/2-language-rules.md#217-should-not-define-case-classes-nested-in-other-classes)
  - [2.18. MUST NOT include classes, traits and objects inside package objects](sections/2-language-rules.md#218-must-not-include-classes-traits-and-objects-inside-package-objects)

- [3. Application Architecture](sections/3-architecture.md)
  - [3.1. SHOULD NOT use the Cake pattern](sections/3-architecture.md#31-should-not-use-the-cake-pattern)
  - [3.2. MUST NOT put things in Play's Global](sections/3-architecture.md#32-must-not-put-things-in-plays-global)
  - [3.3. SHOULD NOT apply optimizations without profiling](sections/3-architecture.md#33-should-not-apply-optimizations-without-profiling)
  - [3.4. SHOULD be mindful of the garbage collector](sections/3-architecture.md#34-should-be-mindful-of-the-garbage-collector)
  - [3.5. MUST NOT use parameterless ConfigFactory.load() or access a Config object directly](sections/3-architecture.md#35-must-not-use-parameterless-configfactoryload-or-access-a-config-object-directly)

- [4. Concurrency and Parallelism](sections/4-concurrency-parallelism.md)
  - [4.1. SHOULD avoid concurrency like the plague it is](sections/4-concurrency-parallelism.md#41-should-avoid-concurrency-like-the-plague-it-is)
  - [4.2. SHOULD use appropriate abstractions only where suitable](sections/4-concurrency-parallelism.md#42-should-use-appropriate-abstractions-only-where-suitable---future-actors-rx)
  - [4.3. SHOULD NOT wrap purely CPU-bound operations in Futures](sections/4-concurrency-parallelism.md#43-should-not-wrap-purely-cpu-bound-operations-in-futures)
  - [4.4. MUST use Scala's BlockContext on blocking I/O](sections/4-concurrency-parallelism.md#44-must-use-scalas-blockcontext-on-blocking-io)
  - [4.5. SHOULD NOT block](sections/4-concurrency-parallelism.md#45-should-not-block)
  - [4.6. SHOULD use a separate thread-pool for blocking I/O](sections/4-concurrency-parallelism.md#46-should-use-a-separate-thread-pool-for-blocking-io)
  - [4.7. All public APIs SHOULD BE thread-safe](sections/4-concurrency-parallelism.md#47-all-public-apis-should-be-thread-safe)
  - [4.8. SHOULD avoid contention on shared reads](sections/4-concurrency-parallelism.md#48-should-avoid-contention-on-shared-reads)
  - [4.9. MUST provide a clearly defined and documented protocol for each component or actor that communicates over async boundaries](sections/4-concurrency-parallelism.md#49-must-provide-a-clearly-defined-and-documented-protocol-for-each-component-or-actor-that-communicates-over-async-boundaries)
  - [4.10. SHOULD always prefer single producer scenarios](sections/4-concurrency-parallelism.md#410-should-always-prefer-single-producer-scenarios)
  - [4.11. MUST NOT hard-code the thread-pool / execution context](sections/4-concurrency-parallelism.md#411-must-not-hardcode-the-thread-pool--execution-context)

- [5. Akka Actors](sections/5-actors.md)
  - [5.1. SHOULD evolve the state of actors only in response to messages received from the outside](sections/5-actors.md#51-should-evolve-the-state-of-actors-only-in-response-to-messages-received-from-the-outside)
  - [5.2. SHOULD mutate state in actors only with "context.become"](sections/5-actors.md#52-should-mutate-state-in-actors-only-with-contextbecome)
  - [5.3. MUST NOT leak the internal state of an actor in asynchronous closures](sections/5-actors.md#53-must-not-leak-the-internal-state-of-an-actor-in-asynchronous-closures)
  - [5.4. SHOULD do back-pressure](sections/5-actors.md#54-should-do-back-pressure)
  - [5.5. SHOULD NOT use Akka FSM](sections/5-actors.md#55-should-not-use-akka-fsm)

---

## Contribute

Open an issue to make suggestions, or create a pull request ;-)

---

Copyright &copy; 2015-2016, Some Rights Reserved.<br />
Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
