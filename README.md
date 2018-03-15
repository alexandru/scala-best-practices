# Scala Best Practices

[![Join the chat at https://gitter.im/alexandru/scala-best-practices](https://badges.gitter.im/alexandru/scala-best-practices.svg)](https://gitter.im/alexandru/scala-best-practices?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="100" height="100" />

A collection of best practices, friendly to people that want to
contribute.

- Version: `1.2`
- Updated at: `2016-06-08`

## Table of Contents

- [0. Preface](sections/0-preface.md)
  - [MUST NOT follow advice blindly](sections/0-preface.md#must-not-follow-advice-blindly)

- [1. Hygienic Rules](sections/1-hygienic-rules.md)
  - [SHOULD enforce a reasonable line length](sections/1-hygienic-rules.md#should-enforce-a-reasonable-line-length)
  - [MUST NOT rely on a SBT or IDE plugin to do the formatting for you](sections/1-hygienic-rules.md#must-not-rely-on-a-sbt-or-ide-plugin-to-do-the-formatting-for-you)
  - [SHOULD break long functions](sections/1-hygienic-rules.md#should-break-long-functions)
  - [MUST NOT introduce spelling errors in names and comments](sections/1-hygienic-rules.md#must-not-introduce-spelling-errors-in-names-and-comments)
  - [Names MUST be meaningful](sections/1-hygienic-rules.md#names-must-be-meaningful)

- [2. Language Rules](sections/2-language-rules.md)
  - [MUST NOT use "return"](sections/2-language-rules.md#must-not-use-return)
  - [SHOULD use immutable data structures](sections/2-language-rules.md#should-use-immutable-data-structures)
  - [SHOULD NOT update a "var" using loops or conditions](sections/2-language-rules.md#should-not-update-a-var-using-loops-or-conditions)
  - [SHOULD NOT define useless traits](sections/2-language-rules.md#should-not-define-useless-traits)
  - [MUST NOT use "var" inside a case class](sections/2-language-rules.md#must-not-use-var-inside-a-case-class)
  - [SHOULD NOT declare abstract "var" members](sections/2-language-rules.md#should-not-declare-abstract-var-members)
  - [MUST NOT throw exceptions for validations of user input or flow control](sections/2-language-rules.md#must-not-throw-exceptions-for-validations-of-user-input-or-flow-control)
  - [MUST NOT catch Throwable](sections/2-language-rules.md#must-not-catch-throwable-when-catching-exceptions)
  - [MUST NOT use "null"](sections/2-language-rules.md#must-not-use-null)
  - [MUST NOT use "Option.get"](sections/2-language-rules.md#must-not-use-optionget)
  - [MUST NOT use Java's Date or Calendar, instead use `java.time` (JSR-310)](sections/2-language-rules.md#must-not-use-javas-date-or-calendar-instead-use-javatime-jsr-310)
  - [SHOULD NOT use Any or AnyRef or isInstanceOf / asInstanceOf](sections/2-language-rules.md#should-not-use-any-or-anyref-or-isinstanceof--asinstanceof)
  - [MUST serialize dates as either Unix Timestamp or ISO 8601](sections/2-language-rules.md#must-serialize-dates-as-either-unix-timestamp-or-as-iso-8601)
  - [MUST NOT use magic values](sections/2-language-rules.md#must-not-use-magic-values)
  - [SHOULD NOT use "var" as shared state](sections/2-language-rules.md#should-not-use-var-as-shared-state)
  - [Public functions SHOULD have an explicit return type](sections/2-language-rules.md#public-functions-should-have-an-explicit-return-type)
  - [SHOULD NOT define case classes nested in other classes](sections/2-language-rules.md#should-not-define-case-classes-nested-in-other-classes)
  - [MUST NOT include classes, traits and objects inside package objects](sections/2-language-rules.md#must-not-include-classes-traits-and-objects-inside-package-objects)
  - [SHOULD use head/tail and init/last decomposition only if they can be done in constant time and memory](sections/2-language-rules.md#should-use-headtail-and-initlast-decomposition-only-if-they-can-be-done-in-constant-time-and-memory)

- [3. Application Architecture](sections/3-architecture.md)
  - [SHOULD NOT use the Cake pattern](sections/3-architecture.md#should-not-use-the-cake-pattern)
  - [MUST NOT put things in Play's Global](sections/3-architecture.md#must-not-put-things-in-plays-global)
  - [SHOULD NOT apply optimizations without profiling](sections/3-architecture.md#should-not-apply-optimizations-without-profiling)
  - [SHOULD be mindful of the garbage collector](sections/3-architecture.md#should-be-mindful-of-the-garbage-collector)
  - [MUST NOT use parameterless ConfigFactory.load() or access a Config object directly](sections/3-architecture.md#must-not-use-parameterless-configfactoryload-or-access-a-config-object-directly)

- [4. Concurrency and Parallelism](sections/4-concurrency-parallelism.md)
  - [SHOULD avoid concurrency like the plague it is](sections/4-concurrency-parallelism.md#should-avoid-concurrency-like-the-plague-it-is)
  - [SHOULD use appropriate abstractions only where suitable](sections/4-concurrency-parallelism.md#should-use-appropriate-abstractions-only-where-suitable---future-actors-rx)
  - [SHOULD NOT wrap purely CPU-bound operations in Futures](sections/4-concurrency-parallelism.md#should-not-wrap-purely-cpu-bound-operations-in-scalas-standard-futures)
  - [MUST use Scala's BlockContext on blocking I/O](sections/4-concurrency-parallelism.md#must-use-scalas-blockcontext-on-blocking-io)
  - [SHOULD NOT block](sections/4-concurrency-parallelism.md#should-not-block)
  - [SHOULD use a separate thread-pool for blocking I/O](sections/4-concurrency-parallelism.md#should-use-a-separate-thread-pool-for-blocking-io)
  - [All public APIs SHOULD BE thread-safe](sections/4-concurrency-parallelism.md#all-public-apis-should-be-thread-safe)
  - [SHOULD avoid contention on shared reads](sections/4-concurrency-parallelism.md#should-avoid-contention-on-shared-reads)
  - [MUST provide a clearly defined and documented protocol for each component or actor that communicates over async boundaries](sections/4-concurrency-parallelism.md#must-provide-a-clearly-defined-and-documented-protocol-for-each-component-or-actor-that-communicates-over-async-boundaries)
  - [SHOULD always prefer single producer scenarios](sections/4-concurrency-parallelism.md#should-always-prefer-single-producer-scenarios)
  - [MUST NOT hard-code the thread-pool / execution context](sections/4-concurrency-parallelism.md#must-not-hardcode-the-thread-pool--execution-context)

- [5. Akka Actors](sections/5-actors.md)
  - [SHOULD evolve the state of actors only in response to messages received from the outside](sections/5-actors.md#should-evolve-the-state-of-actors-only-in-response-to-messages-received-from-the-outside)
  - [SHOULD mutate state in actors only with "context.become"](sections/5-actors.md#should-mutate-state-in-actors-only-with-contextbecome)
  - [MUST NOT leak the internal state of an actor in asynchronous closures](sections/5-actors.md#must-not-leak-the-internal-state-of-an-actor-in-asynchronous-closures)
  - [SHOULD do back-pressure](sections/5-actors.md#should-do-back-pressure)
  - [SHOULD NOT use Akka FSM](sections/5-actors.md#should-not-use-akka-fsm)

---

## Contribute

Open an issue to make suggestions, or create a pull request ;-)

---

Copyright &copy; 2015-2016, Some Rights Reserved.<br />
Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
