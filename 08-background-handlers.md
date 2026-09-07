# Background Handlers

**Concept.** A consumer, a queue handler and a scheduled job are delivery
boundaries, just like a controller. They translate the envelope they receive,
validate the input, delegate to a use-case and let the infrastructure decide
retry and dead-letter.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> @turystack. The split, `XXX-n` versus `XXX-Ln`, and why an `ARC-…` law is
> cited and never restated: `turystack-backend-pattern` › *How a section is
> written*. This section is thin on purpose: almost everything a background
> handler must obey is already constitutional, because "the message arrives
> twice, out of order, and may fail" is not a framework detail.

---

**Rules defined here:** `BGH-2` · `BGH-6` — the law is the *Invariants* table
below; every ❌ item cites the id it violates.

**Retired ids:** `BGH-1` · `BGH-3` · `BGH-4` · `BGH-5` — retired, not
renumbered. A review or commit citing one points at a rule that no longer
exists; the number is never reused.

## 🌐 Generic pattern (portable — stack-independent)

### Invariants (the law the gates enforce)

| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| BGH-2 | The schema validates the payload at the boundary, before any effect | constitutional | `gate:handler-schema` | Shape example / ❌ |
| BGH-6 | A handler owns no cadence: the schedule is the infrastructure rule that invokes it, never a cron, a timer or a loop in the code | constitutional | `grit:no-background-loop` | Background work is its own app / ❌ |

Ids are stable across versions; a gap is a law that moved to the constitution.

**Why BGH-2 exists separately from the controller's schema.** A queue payload
looks trustworthy because it came from inside the system, and that is exactly
why it is not: it was serialized by an older deploy, replayed from a dead-letter
queue, or hand-published during an incident. The envelope proves delivery, never
shape.

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-DEL-1` | The delivery boundary is thin. | the handler validates, delegates, returns |
| `ARC-DEL-3` | The scheduler discovers and distributes. | the `@Handler('SCHEDULE')` app publishes work; it does not process the batch |
| `ARC-DEL-4` | One item's failure does not stop the rest. | per-item error handling inside the batch |
| `ARC-DEL-5` | A failure is rethrown so the infra can retry and dead-letter. | throw out of `execute`; never catch-and-return |
| `ARC-DEL-6` | A batch uses bounded concurrency and an explicit failure policy. | bounded fan-out, never a raw `Promise.all` over a dynamic list |
| `ARC-DEL-7` | Overlapping scheduled execution is a decision. | `@turystack/nestjs-lock` when overlap must not happen |
| `ARC-IDM-1` | Duplicate delivery is assumed. | the use-case is idempotent; `@turystack/nestjs-idempotency` when a key is needed |
| `ARC-IDM-3` | The handler is commutative: an old message after a new one does not corrupt state. | transition decided from `(state, event)`, never from arrival order |
| `ARC-CON-5` | An event is emitted only after the write commits. | publish after the use-case returns |

---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack)

### Background work is its own app

There is one answer, and it is an app in `apps/`. Reacting to an event and
running on a schedule are deliveries, and a delivery that runs inside the API
shares its deploy, its scaling and its failure mode — which is the definition
`ARC-TOP-1` uses to say it should have been its own.

- A handler is an isolated delivery point and uses `@Handler` from
  `@turystack/nestjs-serverless`.
- It imports operations from the domain packages it delivers — `@acme/order`,
  not a copy — and registers only the provider closure that handler needs.
- It owns its `config.schema.ts`; it does not read `process.env` inside the
  handler.
- It calls the same use-case a route would call. The delivery point changed;
  the operation did not.

**A schedule is infrastructure, not code.** The cron expression lives in the
EventBridge Scheduler rule that invokes the function, and the handler declares
`@Handler('SCHEDULE')`. Nothing in the repository holds a cron string, and
nothing keeps a timer alive: `ARC-TOP-5` — a short-lived process freezes after
responding, so a loop started in one dies mid-batch.

The decorators, supported adapters, envelopes and retry options belong to the
`@turystack/nestjs-serverless` documentation.

### Shape example

```typescript
@Handler({ schema: paymentRequestedSchema })
export class ProcessPaymentHandler {
  constructor(
    private readonly processPaymentUseCase: ProcessPaymentUseCase,
  ) {}

  async execute(input: PaymentRequested) {
    return this.processPaymentUseCase.execute(input)
  }
}
```

### ❌ Never do

- `[ARC-DEL-1]` Copying a use-case rule into the handler.
- `[ARC-DEL-1]` Accessing `DatabaseService` or a repository directly.
- `[ARC-DEL-5]` Catching an error and returning success/null — the retry and the dead-letter queue never fire.
- `[ARC-DEL-6]` Running an unbounded `Promise.all` over a dynamic batch.
- `[ARC-LAY-8]` Inventing a wrapper around a queue, schedule, lock or logger the lib already provides.
- `[BGH-2]` Trusting the payload because it came from inside the system.
- `[BGH-6]` Giving a handler its own cadence — a cron string, a `setInterval`, a loop that waits for the next tick. The schedule belongs to the rule that invokes the function, where the deployment can see it.
- `[ARC-IDM-1]` Assuming the message arrives exactly once.
