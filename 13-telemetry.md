# Telemetry Policy

**Concept.** Telemetry exists to explain the system's state and failures without
duplicating noise. The application decides events and business context; the
Turystack libs provide logger, metrics, tracing and provider integration.

## Invariants

| ID | Law |
|---|---|
| TEL-4 | A frequent read uses `debug`; a completed mutation/action uses `info`; an unexpected failure uses `error`. |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-OBS-1` | Correlation per operation. |
| `ARC-OBS-3` | Structured logging. |
| `ARC-OBS-4` | An error is recorded once, at the boundary. |
| `ARC-OBS-5` | Low cardinality. |
| `ARC-OBS-6` | Business metric after success. |
| `ARC-OBS-7` | An alert represents impact. |
| `ARC-10` | Secrets and PII out of telemetry. |
| `ARC-14` | Logger and observability from the libs, directly. |


## Where to record

| Signal | Owner |
|---|---|
| HTTP request, status, duration and error | server/interceptor |
| Consumption, retry and DLQ | handler/serverless boundary |
| Completed business operation | use-case |
| External provider call and latency | adapter or the lib's instrumentation |
| Query and connection | database library |

Do not wrap every public method in `try/catch` just to repeat the same error.
Add a local log only when that layer adds context the boundary cannot
reconstruct.

## Recommended context

Include `operation`, the outcome, the resource's stable type and the
correlation/trace id when available. Identifiers may appear in structured logs
when needed for investigation and allowed by the data policy; they never become
a metric dimension.

The concrete names of decorators, transports, exporters and options belong to
the documentation of `@turystack/nestjs-logger` and
`@turystack/nestjs-observability`.

## Never do

- Logging and swallowing the exception.
- Logging the same failure in the controller, the use-case and the repository.
- Using `organizationId`/`userId` as a metric label.
- Measuring success before the transaction or integration completes.
- Creating `infrastructure/observability`, `support/logger` or a logger wrapper.
