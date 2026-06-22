---
title: Context is Curated, Not Captured
description: Structured logs are only as good as the structure going in. A domain exception model, translation pipeline, and enricher pattern that enforce context consistency at the exception level.
published: true
tags: "architecture, softwareengineering, php, php8"
---

### The Problem Part 1 Didn't Solve

[Part 1](https://dev.to/georgeff/your-php-logs-are-lying-to-you-4g72) gave you consistent log line structure: every entry has a consistent envelope. But the context is still a free-for-all.

Two common anti-patterns:

- Logging raw exception messages: `$logger->error($e->getMessage())`, a string with no structure, no machine-readable context
- Dumping arbitrary data into the context: raw third-party API responses, full request payloads, deeply nested objects thrown into the context array

The second anti-pattern has real operational consequences. Log platforms have entry size limits; oversized context causes truncation or log entries being split and needing recomposition in the platform. Inconsistent field names across the codebase (`userId`, `user_id`, `uid`) destroy the aggregation the platform was bought to provide. The problem isn't the logger; the logger is doing its job. The problem is that the context is treated as a dumping ground rather than a contract.

### Context is Curated, Not Captured

The discipline: define what fields belong on a log entry for a given exception ahead of time, not at the call site. The call site should never decide what context gets logged; the decision belongs at the domain model level.

This is the difference between:

```php
// Captured - call site decides, schema varies everywhere
$logger->error($e->getMessage(), ['response' => $apiResponse, 'user' => $user]);

// Curated - schema defined on the exception, the reporter just delivers it
$logger->log($e->severity->value, $e->getMessage(), $e->structuredData);
```

`structuredData` is a property on the domain exception; it returns the complete structural representation of the exception: error code, severity, retryable flag, occurred_at, context, metadata. The reporter doesn't decide what gets logged; the exception does.

The reporter's job is minimal. Translate the throwable, then call the logger with the three values the domain exception already knows about itself.

```php
$e = $this->translator->translate($exception);

$this->logger->log($e->severity->value, $e->getMessage(), $e->structuredData);
```

Consistency is the goal. Every `PaymentFailedException` that gets logged looks identical in your logging platform, regardless of where in the codebase it was thrown.

### The Domain Exception Model

A domain exception is an exception that carries its own structured context. It knows severity, its error code, whether the operation is retryable, and the specific context field that describes it.

Key properties:

- `severity`: not inferred by the logger, declared by the exception itself. `severity` is a backed enum (`critical`, `error`, `warning`, `info`, `debug`). `$e->severity->value` gives the string the logger expects. Type safety means no invalid severity values, no typos making it into production.
- `context`: curated key/value data defined at construction, not at the log call.
- `retryable`: operational metadata that belongs on the log entry.
- `occurredAt`: RFC 3339 timestamp captured at throw time, not log time.
- `getErrorCode()`: structured error code (e.g. `DB_0001`, `PAYMENT_0033`) that is independently queryable in your log platform.
- `structuredData`: the complete structured representation of the exception, ready for the logger. Uses a property hook (PHP 8.4+), computed once, then cached.

Asymmetric property visibility `public protected(set)` (PHP 8.4+). Properties are publicly readable but only settable within the class and its subclasses. Immutable from the outside; prevents mutation after construction. `getErrorCode()` is abstract; every concrete exception must declare its own code. This is the contract that enforces the structured error code pattern across the codebase.

As a fallback for exceptions that have not been modeled yet, an `UnknownException` is defined that captures the original class, code, file and line. `UNKNOWN_0000` as the error code is a signal in your log platform that something needs modeling.

### The Translation Pipeline

Not every exception in your application will be a `DomainException`. Third-party libraries throw their own exceptions, PHP itself throws runtime errors. The translator's job is to accept any `Throwable` and return a `DomainException`.

The `TranslationHandler` contract has three methods.

```php
interface TranslationHandler
{
    public function matches(Throwable $exception): bool;
    public function handle(Throwable $exception): DomainException;
    public function priority(): int;
}
```

- `matches()`: does the handler know how to translate this exception?
- `handle()`: translate the `Throwable` into a `DomainException`.
- `priority()`: handlers are sorted once during construction, not at call time. Higher priority wins.

The translator's full constructor:

```php
public function __construct(TranslationHandler ...$handlers)
{
    $sorted = [...$handlers];
    usort($sorted, fn($a, $b) => $b->priority() <=> $a->priority());
    $this->handlers = $sorted;
}
```

This is the same principle as the boot/run boundary from the [kernel](https://dev.to/georgeff/two-phases-two-apis-425j). Expensive work (sorting) happens once at construction; the hot path (`translate()`) just iterates an already-sorted list.

The translator walks the already sorted handler list and finds the first match.

```php
$handler = array_find(
    $this->handlers,
    fn($h, $_) => $h->matches($exception)
);
```

`array_find` (PHP 8.4+) is the right tool. Find the first match and stop. No manual loops, no breaks; the intent is immediately readable.

The full pipeline: every `Throwable` that enters `translate()` follows exactly one path:

**Step 1: already a domain exception, return as is**

```php
if ($exception instanceof DomainException) {
    return $exception;
}
```

No translation needed, no information lost.

**Step 2: find the first matching handler**

```php
$handler = array_find(
    $this->handlers,
    fn($h, $_) => $h->matches($exception)
);
```

**Step 3: no handler matched, wrap in an `UnknownException` and return**

```php
if (null === $handler) {
    return new UnknownException($exception);
}
```

Nothing is swallowed. `UNKNOWN_0000` in the platform is a signal that this exception type needs a handler written for it.

**Step 4: handler matched, translate and return**

```php
return $handler->handle($exception);
```

The handler produces a `DomainException` with curated context.

The pipeline is extensible. Add a handler for any third-party exception type and it gets curated context without touching the call site.

### The Enricher Pattern

The enricher sits above the `DomainException`. It adds context fields to every log entry regardless of what is being logged. The `ContextEnricher` contract defines a single method: `enrich(array $context): array`. Enrichers are composed into the logger via the decorator pattern described in [Part 1](https://dev.to/georgeff/your-php-logs-are-lying-to-you-4g72), wired at boot, invisible to the call site. The addition operator preserves caller-set values: `$context + ['field' => $value]`. An enricher never overwrites context that the `DomainException` already defined.

A concrete example: `CorrelationIdEnricher`

```php
public function enrich(array $context): array
{
    return $context + ['correlation_id' => (string) $this->correlationId];
}
```

Every log entry now carries a `correlation_id` field, added once at the composition root, without a single call site change. The enricher is registered by the `StructuredLoggingModule` and auto-resolved by the kernel tag registry; the developer adds the module explicitly, and the enricher wires itself through the tag system with no further call site work. What `correlation_id` is, where it comes from, and how it connects log entries across service boundaries will be explained in Part 3.

### What You Have Now and What's Still Missing

Every log entry has a consistent envelope ([Part 1](https://dev.to/georgeff/your-php-logs-are-lying-to-you-4g72)), every exception produces consistent curated structured context, and every log entry carries a `correlation_id` field. Your log platform can now aggregate by error code, filter by severity, track retryable vs non-retryable failures, and report on error rates by type. But, `correlation_id` is only useful if it's the same value across all log entries for a given request, and across all services that handle that request. A `correlation_id` that resets per service tells you nothing about the flow of the request through a distributed system.

Part 3 of this series will define what correlation IDs are, why they matter, and how they flow across service boundaries.

---

If you want to see a full concrete implementation of everything covered in this article, `meritum/structured-logging` is a domain exception model, translation pipeline, and enricher system built on these exact principles.

- [github.com/MeritumIO/structured-logging](https://github.com/MeritumIO/structured-logging)
- [packagist.org/packages/meritum/structured-logging](https://packagist.org/packages/meritum/structured-logging)

---

Natural Aristoi - [dev.to/georgeff/natural-aristoi-h6j](https://dev.to/georgeff/natural-aristoi-h6j)
Two Phases, Two APIs - [dev.to/georgeff/two-phases-two-apis-425j](https://dev.to/georgeff/two-phases-two-apis-425j)
