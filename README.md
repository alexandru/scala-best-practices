# Scala Best Practices

A collection of best practices, friendly to people that want to
contribute.

- Version: `0.1`
- Updated at: `2014-10-20`

## Table of Contents

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

- [0. Preface](0-preface.md)
  - [0.1 MUST NOT follow advice blindly](0-preface.md#01-must-not-follow-advice-blindly)

- [1. General Advice](1-general-advice.md)
  - [1.1. MUST NOT use "return"](1-general-advice.md#11-must-not-use-return)
  - [1.2. SHOULD use immutable data structures](1-general-advice.md#12-should-use-immutable-data-structures)
  - [1.3. SHOULD NOT update a "var" using loops or conditions](1-general-advice.md#13-should-not-update-a-var-using-loops-or-conditions)
  - [1.4. SHOULD NOT define useless traits](1-general-advice.md#14-should-not-define-useless-traits)
  - [1.5. MUST NOT use "var" inside a case class](1-general-advice.md#15-must-not-use-var-inside-a-case-class)
  - [1.6. SHOULD NOT declare abstract val or var or lazy val members](1-general-advice.md#16-should-not-declare-abstract-val-or-var-or-lazy-val-members)
  - [1.7. MUST NOT throw exceptions for validations of user input or flow control](1-general-advice.md#17-must-not-throw-exceptions-for-validations-of-user-input-or-flow-control)
  - [1.8. MUST NOT catch Throwable](1-general-advice.md#18-must-not-catch-throwable-when-catching-exceptions)
  - [1.9. MUST NOT use "null"](1-general-advice.md#19-must-not-use-null)
  - [1.10. MUST NOT use Java's Date or Calendar, instead use Joda-Time](1-general-advice.md#110-must-not-use-javas-date-or-calendar-instead-use-joda-time)
  - [1.11. SHOULD NOT use Any or AnyRef or isInstanceOf / asInstanceOf](1-general-advice.md#111-should-not-use-any-or-anyref-or-isinstanceof--asinstanceof)
  - [1.12. MUST serialize dates as either Unix Timestamp or ISO 8601](1-general-advice.md#112-must-serialize-dates-as-either-unix-timestamp-or-as-iso-8601)
  - [1.13. MUST NOT use magic values](1-general-advice.md#113-must-not-use-magic-values)
  - [1.14. SHOULD NOT use var as shared state](1-general-advice.md#114-should-not-use-var-as-shared-state)

- [2. Application Architecture](2-architecture.md)
  - [2.1. SHOULD NOT use the Cake pattern](2-architecture.md#21-should-not-use-the-cake-pattern)
  - [2.2. MUST NOT put things in Play's Global](2-architecture.md#22-must-not-put-things-in-plays-global)

- [3. Concurrency, Parallelism and Performance](3-concurrency-paralelism-performance.md)
  - [3.1. SHOULD avoid concurrency like the plague it is](3-concurrency-paralelism-performance.md#31-should-avoid-concurrency-like-the-plague-it-is)
  - [3.2. SHOULD use appropriate abstractions only where suitable](3-concurrency-paralelism-performance.md#32-should-use-appropriate-abstractions-only-where-suitable---future-actors-rx)
  - [3.3. MUST NOT wrap purely CPU-bound operations in Futures](3-concurrency-paralelism-performance.md#33-must-not-wrap-purely-cpu-bound-operations-in-futures)
  - [3.4. MUST use Scala's BlockContext on blocking I/O](3-concurrency-paralelism-performance.md#34-must-use-scalas-blockcontext-on-blocking-io)
  - [3.5. SHOULD NOT block](3-concurrency-paralelism-performance.md#35-should-not-block)
  - [3.6. All public APIs SHOULD BE thread-safe](3-concurrency-paralelism-performance.md#36-all-public-apis-should-be-thread-safe)
  - [3.7. SHOULD avoid contention on shared reads](3-concurrency-paralelism-performance.md#37-should-avoid-contention-on-shared-reads)
  - [3.8. MUST evolve the state of actors only in response to messages received from the outside](3-concurrency-paralelism-performance.md#38-must-evolve-the-state-of-actors-only-in-response-to-messages-received-from-the-outside)
  - [3.9. MUST provide a clearly defined and documented protocol for each component or actor that communicates over async boundaries](3-concurrency-paralelism-performance.md#39-must-provide-a-clearly-defined-and-documented-protocol-for-each-component-or-actor-that-communicates-over-async-boundaries)
  - [3.10. SHOULD avoid having mutable state in persistent actors](3-concurrency-paralelism-performance.md#310-should-avoid-having-mutable-state-in-persistent-actors)
  - [3.11. SHOULD always prefer single producer scenarios](3-concurrency-paralelism-performance.md#311-should-always-prefer-single-producer-scenarios)
  - [3.12. MUST be mindful of the garbage collector and how unnecessary allocations can kill performance](3-concurrency-paralelism-performance.md#312-must-be-mindful-of-the-garbage-collector-and-how-unnecessary-allocations-can-kill-performance)
  - [3.13. MUST NOT hardcode the thread-pool / execution context](3-concurrency-paralelism-performance.md#313-must-not-hardcode-the-thread-pool--execution-context)

## Contribute

Submit a PR.

---

Copyright &copy; 2014, Some Rights Reserved.<br />Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
