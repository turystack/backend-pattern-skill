# Telemetry Policy

**Concept.** Telemetry exists to explain the system's state and failures without
duplicating noise. The application decides events and business context; the
Turystack libs provide logger, metrics, tracing and provider integration.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> @turystack. The split, `XXX-n` versus `XXX-Ln`, and why an `ARC-…` law is
> cited and never restated: `turystack-backend-pattern` › *How a section is
> written*. Observability is almost entirely constitutional, so this file is
> mostly a placement guide.

---

**Rules defined here:** `TEL-4` — the law is the *Invariants* table below;
every ❌ item cites the id it violates.

**Retired ids:** `TEL-1` · `TEL-2` · `TEL-3` — retired, not renumbered. A
review or commit citing one points at a rule that no longer exists; the number
is never reused.

## 🌐 Generic pattern (portable — stack-independent)

### Invariants (the law the gates enforce)

| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| TEL-4 | A frequent read logs at `debug`; a completed mutation/action at `info`; an unexpected failure at `error` | constitutional | `grit:log-level` | Where to record / ❌ |

Ids are stable across versions; a gap is a law that moved to the constitution.

**Why the level is a law and not taste.** The level is what decides whether a
signal survives production sampling. A read logged at `info` drowns the
mutations someone will search for during an incident, and a failure logged at
`warn` never reaches the alert that represents impact (`ARC-OBS-7`).

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-OBS-1` | Every operation carries a correlation identifier. | `@turystack/nestjs-context` propagates it from the edge |
| `ARC-OBS-2` | The correlation crosses process boundaries. | the publisher sends it; the handler restores it |
| `ARC-OBS-3` | Logs are structured: stable message plus fields. | `logger.info('order.cancelled', { orderId })`, never interpolation |
| `ARC-OBS-4` | An error is logged once, at the highest boundary. | the exception filter logs; inner layers rethrow |
| `ARC-OBS-5` | Dimensions are low cardinality. | an identifier may be a log field, never a metric label |
| `ARC-OBS-6` | A business metric is emitted after confirmed success. | after commit, not before the transaction |
| `ARC-OBS-7` | An alert represents actionable impact. | alert on the operation's failure rate, not on one exception |
| `ARC-OBS-8` | Instrumentation never changes behavior. | a telemetry failure never fails the operation |
| `ARC-SEC-7` | Secrets and PII stay out of telemetry. | no `password`, `token` or PII field in a log or span |
| `ARC-LAY-8` | Infra covered by a lib is used directly. | `@turystack/nestjs-logger` / `@turystack/nestjs-observability` injected |

---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack)

### Where to record

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

### Recommended context

Include `operation`, the outcome, the resource's stable type and the
correlation/trace id when available. Identifiers may appear in structured logs
when needed for investigation and allowed by the data policy; they never become
a metric dimension.

The concrete names of decorators, transports, exporters and options belong to
the documentation of `@turystack/nestjs-logger` and
`@turystack/nestjs-observability`.

### ❌ Never do

- `[ARC-ERR-7]` Logging and swallowing the exception.
- `[ARC-OBS-4]` Logging the same failure in the controller, the use-case and the repository.
- `[ARC-OBS-5]` Using `organizationId`/`userId` as a metric label.
- `[ARC-OBS-6]` Measuring success before the transaction or integration completes.
- `[ARC-LAY-8]` Creating `infrastructure/observability`, `support/logger` or a logger wrapper.
- `[ARC-OBS-3]` Interpolating context into the message string instead of passing fields.
- `[ARC-SEC-7]` Logging a request body that still carries a password, token or PII.
- `[TEL-4]` Logging a list read at `info`, or a failure at `warn`.
