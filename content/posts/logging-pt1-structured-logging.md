---
title: Your PHP Logs are Lying to You
description: Plain JSON to stdout and a PSR-3 interface. The simplest PHP logging setup is also the most useful one.
published: true
tags: 'architecture, softwareengineering, php, php8'
id: 3916791
date: '2026-06-16T14:42:04Z'
---

### The Evolution: From Log Tails to Indexed Search

Most teams start with a stream-based log platform, such as Papertrail, Loggly, or legacy syslog viewers. These platforms are fast to set up, and work well early on, but ultimately are just live tail with grep. Unstructured logs work fine here because a human is reading and searching them manually. As the application and team grow, you graduate to an indexed search platform, such as Elastic/OpenSearch, Datadog Logs, or Loki with LogQL. This is where unstructured logs break down because these platforms expect to index fields, not parse free text. You can still ship plain text to Elastic, but you've thrown away everything that makes it powerful: field filtering, aggregations, dashboards, and alerting on specific values. If you're thinking "we write to a log file, not a stream", that's fine, and many of these stream platforms are reading a file anyway. The live tail is literally `tail -f`. The problem was never the destination. The problem is what the file contains. The platform didn't change your logs, it exposed what your logs already were: unstructured text. Structured logs enable trend analysis: error rate over time, errors by application version, change failure rate after a deployment. These are questions grep can't answer. "Did our error rate increase after v2.3.0 shipped?" requires `version`, `level`, and `timestamp` as discrete, indexed fields you can aggregate on, not substrings buried in formatted strings. These are the kinds of questions that matter at an engineering org level. Change failure rate is a DORA metric. Teams measuring engineering effectiveness need their logs to support it, and that's only possible with consistent structure.

### Stdout Decouples Your Application from the Log Destination

Writing to stdout is an intentional architectural boundary, not a lack of a "real" logging setup. Your application's only concern is emitting a well-formed log entry to the stream. Where it goes after that is not the application's problem. In a Kubernetes environment, an operator (Fluentd, Fluent Bit, Logstash, Vector) captures the stdout stream from each container and routes it. Swapping log platforms is an operator configuration change. Zero application code is touched, and there is zero redeployment of your software. A Kubernetes operator can fan logs out to multiple destinations simultaneously: Elastic for search indexing, S3 for long-term archival, PagerDuty for critical-level entries. Your application emits once, and the operator handles the rest. This fan-out is something a tightly coupled or file-based logger cannot do cleanly. The application would have to be aware of every destination. The 12-factor principle formalizes this: treat logs as event streams, let the execution environment handle routing and storage.

### What Structured Logging Actually Is

Each log entry is a self-describing data structure, not a formatted string. Use NDJSON (Newline Delimited JSON), one JSON object per line, independently parseable by the logging platform. The platform indexes each field, not the full string. This is the difference between "find me logs containing the word payment" and "find me all error-level logs where amount > 100 in the last 15 minutes". The platform can only be as powerful as the structure you give it.

Example comparison:

```
# Unstructured
[2026-06-09 14:32:01] ERROR: Payment failed for user 4321

# Structured
{"timestamp":"2026-06-15T10:28:01.123+00:00","level":"error","severity":"error","message":"Payment failed","context":{"user_id":4321,"app_version":"1.3.0"}}
```

What belongs on every log line:

- `timestamp`: RFC 3339 extended, UTC (e.g. `2026-06-10T14:32:01.123+00:00`)
- `level`: original PSR-3 level name, all 8 values are preserved
- `severity`: reduced 5-value set (`debug`, `info`, `warning`, `error`, `critical`) aligned with structured log sinks
- `message`: human-readable summary, kept short
- `context`: structured key/value object, always present even when no context is supplied

Why `level` and `severity`? PSR-3 defines 8 levels, but most sinks don't natively understand all of them. `severity` gives the platform a normalized field it can reliably aggregate on without knowing PSR-3's model. In short, `level` preserves PSR-3 fidelity, `severity` serves the platform. No information is lost and the sink gets what it needs. The practical side effect is the `severity` field ends the "when do I use `emergency` vs `alert` vs `critical`" debate. Those distinctions came from syslog and don't map cleanly to application logging. Your alerting dashboard runs on `severity`. Operationally, all three mean the same thing. The nuance lives in `level` for whoever needs it.

Consistency matters more than completeness. Every line must have the same shape. A `version` field that appears on 60% of log lines is useless for change failure rate reporting.

### PSR-3 and Why It Matters

PSR-3 is the PHP standard logging contract defining 8 severity methods and `log()`. Context, the second argument defined by the interface, is where the structure lives.

```php
$logger->error('Payment failed', ['user_id' => 4321, 'app_version' => '1.3.0']);
```

`user_id` and `app_version` will appear on every "Payment failed" log, providing the logging platform consistent data to index.

Coding to PSR-3 means your application is decoupled from the concrete logger implementation. The decorator pattern is a natural extension of PSR-3 decoupling. Because `LoggerInterface` is an interface, you can wrap any logger that implements the same contract. A context-enriching logger accepts a `LoggerInterface`, enriches the context, then delegates to the inner logger. The call site never changes.

```php
<?php

final class ContextEnrichingLogger implements LoggerInterface
{
    /**
     * @var ContextEnricher[]
     */
    private readonly array $enrichers;

    public function __construct(
        private readonly LoggerInterface $inner,
        ContextEnricher ...$enrichers
    ) {
        $this->enrichers = $enrichers;
    }

    public function info(string $message, array $context = []): void
    {
        $enrichedContext = $this->enrichContext($context);

        $this->inner->info($message, $enrichedContext);
    }

    // Remaining LoggerInterface methods
}
```

This is where consistent context attributes get added once, at the composition root, rather than at every log call site. Wire the decorator at [boot time](https://dev.to/georgeff/two-phases-two-apis-425j) and every log call in the application gets the enriched context automatically. The enrichers can globally capture values such as `app_version`, `environment`, and `correlation_id`. This provides a central location to build out log context, eliminating repetition at every log call. What the enricher pipeline doesn't solve is converting a thrown exception into structured context.

### Writing a Minimal Structured Logger

When writing a minimal logger, implement the `LoggerInterface` as a first-class citizen of the implementation, not an afterthought. The [`psr/log`](https://packagist.org/packages/psr/log) package also ships with a trait defining the 8 standard methods, allowing the implementation to focus on `log()`.

```php
<?php

use Psr\Log\LoggerTrait;
use Psr\Log\LoggerInterface;

final class Logger implements LoggerInterface
{
    use LoggerTrait;

}
```

Write the output to stdout.

```php
/**
 * @var resource
 */
private $output;

public function __construct(mixed $output = STDOUT)
{
    if (!is_resource($output)) {
        throw new \InvalidArgumentException('Logger output must be a resource');
    }

    $this->output = $output;
}
```

Format each entry as a JSON object.

```php
public function log(string $level, \Stringable|string $message, array $context = []): void
{
    $severityMap = [
        'emergency' => 'critical',
        'alert'     => 'critical',
        'critical'  => 'critical',
        'error'     => 'error',
        'warning'   => 'warning',
        'notice'    => 'info',
        'info'      => 'info',
        'debug'     => 'debug',
    ];

    $logEntry = [
        'timestamp' => (new \DateTimeImmutable())->format(\DateTimeInterface::RFC3339_EXTENDED),
        'level'     => $level,
        'severity'  => $severityMap[$level],
        'message'   => (string) $message,
        'context'   => $context,
    ];

    $json = json_encode($logEntry, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE | JSON_THROW_ON_ERROR);

    fwrite($this->output, $json . PHP_EOL);
}
```

Keep it simple. No file rotation, no handlers, no formatters. The logger's only job is to serialize a log entry and write it to the stream.

### What You Have Now and What's Still Missing

You have clean, machine-readable, streamable logs that log aggregators can ingest and index without configuration. Once your logs are structured, how do you enrich them with the proper context, especially when the event being logged is an exception? And that's before we even get to the thousands of JSON lines with no way to know which lines belong to the same request.

Part 2 of this series will cover the domain exception model and translation pipeline that turns exceptions into structured log data.

---

If you want to see a full concrete implementation of everything covered in this article, `meritum/logger` is a minimal PSR-3 structured logger built on these exact principles: RFC 3339 timestamps, NDJSON to stdout, severity normalization for structured log sinks.

- [github.com/MeritumIO/logger](https://github.com/MeritumIO/logger)
- [packagist.org/packages/meritum/logger](https://packagist.org/packages/meritum/logger)
