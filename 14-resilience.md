# Resilience

**Concept.** Every call that leaves the process can hang, fail transiently, or
stay broken for ten minutes. The constitution says each of those has a declared
answer; this section says **where the answer lives in a Turystack backend** and
how the three mechanisms compose without multiplying each other.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> `@turystack/nestjs-resilience`. The split, `XXX-n` versus `XXX-Ln`, and why
> an `ARC-…` law is cited and never restated: `turystack-backend-pattern` ›
> *How a section is written*.

---

**Rules defined here:** `RSL-1` · `RSL-2` · `RSL-3` · `RSL-4` · `RSL-5` — the
law is the *Invariants* table below; every ❌ item cites the id it violates.

## 🌐 Generic pattern (portable — stack-independent)

**RSL-1 — the resilience decision lives at the port, not at the caller.**

Timeout, retry and breaker are properties of **the dependency**, not of whoever
happens to call it today. They are declared once, where the process talks to the
outside — the adapter or the library client — so the second and third callers
inherit them instead of each inventing their own. A use-case that wraps its own
call in a retry has forked the policy. **[RSL-1]**

**RSL-2 — an inner timeout is shorter than the budget that contains it.**

A request has a total budget. Everything it calls has to fit inside what is
left, or the outer boundary gives up while the inner call is still politely
waiting. Write the arithmetic down: if the caller allows 3s and the operation
makes two sequential calls, neither may be allowed 3s. **[RSL-2]**

**RSL-3 — retry happens at exactly one level.**

Attempts multiply when they nest. Three attempts around a client that already
retries three times is nine calls against a dependency that is probably already
struggling, and a total latency nobody budgeted. Pick the level that owns the
retry — normally the port (`RSL-1`) — and the levels above it propagate the
failure instead of trying again. **[RSL-3]**

**RSL-4 — the breaker is named after the dependency, not the method.**

A circuit exists to stop hammering a sick provider. If every method opens its
own circuit, the provider gets N times the failing traffic before any of them
trip, and the first method to recover tells you nothing about the rest. One
circuit per dependency, shared by every call into it. **[RSL-4]**

**RSL-5 — degradation is a product decision, written per operation.**

"What happens when this dependency is down" is not answered by the retry
policy. Each operation states whether it can serve without the dependency (serve
a stale replica, skip an enrichment, queue the effect) or whether it must fail.
Unstated means fail — but it must be a decision, not an accident. **[RSL-5]**

### Invariants (the law the gates enforce)

| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| RSL-1 | Timeout, retry and breaker are declared at the port, once per dependency, never at the call site | constitutional | `gate:resilience-at-port` | Mechanisms / ❌ |
| RSL-2 | An inner timeout is strictly shorter than the budget containing it | constitutional | `manual` | Budget / ❌ |
| RSL-3 | Retry happens at exactly one level of the call stack | constitutional | `gate:no-nested-retry` | Mechanisms / ❌ |
| RSL-4 | One circuit per dependency, named after it — never one per method | constitutional | `grit:breaker-named` | Mechanisms / ❌ |
| RSL-5 | Every operation over a fallible dependency states whether it degrades or fails | constitutional | `manual` | Degradation |

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-RES-1` | Every call leaving the process has a timeout. | `@Timeout(ms)` on the port method |
| `ARC-RES-2` | Retry only over potentially transient failure. | the default `retryOn` already refuses 4xx `AppError`s |
| `ARC-RES-3` | Retry uses backoff with jitter and an attempt ceiling. | `@Retry({ attempts, backoff, jitter, maxDelay })` |
| `ARC-RES-4` | Retry requires idempotency. | the retried method is idempotent, or carries a key (`15-idempotency.md`) |
| `ARC-RES-5` | Exhausted attempts go to a permanent-failure strategy. | rethrow so the consumer dead-letters (`ARC-DEL-5`) |
| `ARC-RES-6` | A degraded dependency fails fast instead of consuming the caller. | `@CircuitBreaker({ name, failureThreshold, resetTimeout })` |
| `ARC-RES-7` | Degradation is decided per operation. | `RSL-5`, written in the use-case |
| `ARC-RES-8` | Timeout, retry and degradation are expected and observable failures. | they surface as catalogue errors and metrics, never as silent `null` |
| `ARC-IDM-7` | An operation with retry is idempotent. | the pairing rule behind `ARC-RES-4` |
| `ARC-OBS-8` | Instrumentation never changes behavior. | breaker state is reported, never used to fail a healthy call |

---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack/nestjs-resilience)

> Options, defaults and composition order are documented by the library. This
> section decides **what to apply and where**; it does not restate the API.

### Mechanisms per rule

- **`ARC-RES-1` / RSL-1** — `@Timeout(ms)` on the adapter method that performs
  the call. Not on the use-case, not on the controller: those are boundaries
  around your own code, and a timeout there hides which dependency was slow.
- **`ARC-RES-3` / RSL-3** — `@Retry({ attempts: 3, backoff: 'exponential', baseDelay: 100, jitter: true, maxDelay: 5_000 })`
  on the same port method. The defaults already express the law; pass options
  when the provider's documented limits differ, not by taste.
- **`ARC-RES-2`** — the default `retryOn` refuses to retry an `AppError` with a
  4xx status, which is exactly the law. Override it only for a provider that
  answers with a wrong status, and say so in the argument.
- **`ARC-RES-6` / RSL-4** — `@CircuitBreaker({ name: 'stripe' })`. The `name` is
  the **dependency**, shared by every method that talks to it; leaving it to the
  default produces `ClassName.methodName`, which is one circuit per method and
  breaks `RSL-4`. `circuitState(name)` reports; `resetCircuits()` belongs to
  tests, never to request code.

### Budget

Write the arithmetic before the decorators (`RSL-2`):

```text
HTTP request budget                    3000 ms
├── provider call        @Timeout(800)  ×3 attempts, backoff ≈ 100 + 200
│                                      ≈ 2700 ms worst case  ← already at the edge
└── own persistence                     the rest

→ either @Timeout(500) with 3 attempts, or 800 ms with 2. Both are decisions;
  neither is "whatever the default was".
```

The trap is silent: each part looks reasonable, the sum does not, and the
symptom is a gateway timeout that blames the wrong service.

### Shape

```typescript
@Injectable()
export class StripeAdapter implements PaymentPort {
  @CircuitBreaker({ name: 'stripe' })
  @Retry({ attempts: 3 })
  @Timeout(800)
  async charge(input: ChargeInput) {
    return this.client.charges.create(input)
  }
}
```

The use-case injects `PaymentPort` and never knows any of it exists — which is
the point of `RSL-1` and of `ARC-LAY-8`.

### Degradation

`RSL-5` is written where the operation lives, not in the adapter:

| Operation | Dependency down | Decision |
|---|---|---|
| Show order detail | enrichment API | serve without it; the section renders its own error state (`ARC-ERR-8`) |
| Create charge | payment provider | fail; there is no partial charge |
| Send receipt | mail provider | queue the effect; the write already committed (`ARC-CON-1`) |

### ❌ Never do

```typescript
// ❌ [RSL-1] policy at the call site — the next caller invents its own
async execute(input: PayOrderInput) {
  return retry(() => this.stripe.charge(input), { attempts: 5 })
}

// ❌ [RSL-3] retry around a client that already retries — 3 × 3 = 9 calls
@Retry({ attempts: 3 })
async charge(input: ChargeInput) {
  return this.clientThatAlreadyRetries.charge(input)
}

// ❌ [RSL-4] one circuit per method — the provider takes N× the failing traffic
@CircuitBreaker() // defaults to StripeAdapter.charge
async charge() {}
@CircuitBreaker() // and a second, unrelated circuit for the same provider
async refund() {}

// ❌ [ARC-RES-1] a call leaving the process with no timeout at all
async charge(input: ChargeInput) {
  return this.client.charges.create(input)
}

// ❌ [ARC-RES-4] retrying a non-idempotent effect — attempt 2 charges again
@Retry({ attempts: 3 })
async charge(input: ChargeInput) { /* no idempotency key */ }

// ❌ [ARC-RES-8] swallowing the failure into a shape the caller cannot tell apart
catch { return null } // "no data" and "the provider is down" are not the same
```
