# Idempotency

**Concept.** The constitution assumes duplicate delivery everywhere. This
section answers the two questions that follow in a Turystack backend: **which
fact becomes the key**, and **what a repeat is supposed to return** — because a
queue handler and an HTTP caller want opposite answers.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> `@turystack/nestjs-idempotency`. The split, `XXX-n` versus `XXX-Ln`, and why
> an `ARC-…` law is cited and never restated: `turystack-backend-pattern` ›
> *How a section is written*.

---

**Rules defined here:** `IDP-1` · `IDP-2` · `IDP-3` · `IDP-4` · `IDP-5` — the
law is the *Invariants* table below; every ❌ item cites the id it violates.

## 🌐 Generic pattern (portable — stack-independent)

**IDP-1 — the key comes from the fact, and which fact depends on the entrypoint.**

`ARC-IDM-2` says the key is stable and born with the fact. In a backend there
are exactly three shapes of fact, and each one already carries its key:

```text
HTTP request      the caller's idempotency key      it owns the retry, so it owns the key
queue message     the envelope's identifier         the producer already named the fact
scheduled run     the window being processed        "invoices of 2026-08-15", not "now"
```

A scheduler that keys on the current instant has no key at all: two runs of the
same window are two different keys, which is exactly the duplicate the law was
meant to stop. **[IDP-1]**

**IDP-2 — a repeat returns what its caller can use.**

Someone waiting on a response needs the *same response*, or the retry looks like
a different outcome. Nothing is waiting on a queue handler, so recording that
the work happened is enough and storing the result is dead weight. Choosing
wrong is not a performance detail: replaying nothing to an HTTP caller turns a
successful retry into an empty success. **[IDP-2]**

**IDP-3 — the key outlives every redelivery that can reach it.**

The window a key is remembered for has to be longer than the longest retry of
whatever delivers the message. A key remembered for an hour, behind a queue that
dead-letters after a day, is a duplicate waiting for a slow incident. Payments
and webhooks are measured in days. **[IDP-3]**

**IDP-4 — the same key with different arguments is a bug, and it fails loudly.**

A retry repeats a request; it does not change it. When a key arrives a second
time carrying different input, something upstream reused a key it should not
have, and the safe answer is to refuse — never to run the new arguments, and
never to silently replay the old result as if it answered them. **[IDP-4]**

**IDP-5 — the guard and the key have different lifetimes on purpose.**

Two problems hide under one word. **Repetition** is a message arriving twice an
hour apart — solved by remembering the key. **Concurrency** is the same message
being processed twice *right now* — solved by a guard around the in-flight
execution. Giving the guard the key's lifetime blocks every legitimate retry
until it expires. **[IDP-5]**

### Invariants (the law the gates enforce)

| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| IDP-1 | The key derives from the fact: caller key, envelope identifier or processed window — never the clock | constitutional | `grit:no-ambient-clock` | Mechanisms / ❌ |
| IDP-2 | A repeat replays the stored result when a caller waits, and skips when none does | constitutional | `manual` | Mechanisms |
| IDP-3 | The key's retention outlives the longest redelivery that can reach it | constitutional | `manual` | Lifetimes / ❌ |
| IDP-4 | Same key with different arguments is refused, never run and never silently replayed | constitutional | `gate:idempotency-fingerprint` | ❌ |
| IDP-5 | The in-flight guard and the remembered key have separate lifetimes | constitutional | `manual` | Lifetimes / ❌ |

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-IDM-1` | Every asynchronous operation assumes duplicate delivery. | the handler is idempotent by construction, or carries a key |
| `ARC-IDM-2` | The deduplication key is stable and comes from the fact. | `IDP-1` names the fact per entrypoint |
| `ARC-IDM-3` | The handler is commutative. | the transition reads current state; late messages cannot regress it |
| `ARC-IDM-4` | The transition is a function of `(state, event)`. | no branch reads arrival order or a counter |
| `ARC-IDM-5` | An event carries absolute state, never a delta. | published payloads state the result, not the change |
| `ARC-IDM-6` | A repeatable request carries an idempotency key when repeating it has an effect. | `@Idempotent` on the operation |
| `ARC-IDM-7` | An operation with retry is idempotent. | pairs with `14-resilience.md` › `ARC-RES-4` |
| `ARC-DEL-5` | A failure is rethrown so the infrastructure retries. | the guard releases; the key is not burned by a failure |
| `ARC-TST-5` | A handler has a duplicate case **and** an out-of-order case. | both are required tests (`12-testing.md` › `TST-8`) |

---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack/nestjs-idempotency)

> Registration, storage and option defaults belong to the library's
> documentation. This section decides **when to reach for it and with what key**.

### Mechanisms per rule

- **`ARC-IDM-6` / IDP-1** — `@Idempotent(resolver, options)` on the operation.
  The resolver is where `IDP-1` becomes code: it reads the key out of the
  arguments (`(args) => args[0].idempotencyKey`, `(args) => args[0].messageId`,
  `(args) => 'invoices:' + args[0].window`) and never out of the clock.
- **IDP-2** — `mode: 'replay'` when an HTTP caller is waiting for a body;
  `mode: 'skip'` for a queue handler, where the repeat resolving to `undefined`
  is the correct answer. `'replay'` is the default because the wrong default for
  a handler wastes storage, while the wrong default for a caller returns an
  empty success.
- **IDP-4** — the library fingerprints the arguments the key was first used
  with and refuses a mismatch. Do not defeat it by hashing only part of the
  input "to be lenient": leniency here means running someone else's payload
  under a key that already answered.
- **IDP-5** — `ttl` remembers the key, `lockTtl` guards the in-flight run.
  Different problems, different lifetimes.
- Cross-cutting operations that are not a single method use
  `IdempotencyService.run` directly with the same key discipline.

### Lifetimes

The three numbers have three different jobs, and two of them are not in the same
unit — read them carefully, because the mistake compiles:

| Option | Unit | Answers | Sized by |
|---|---|---|---|
| `ttl` | **seconds** | how long a repeat is recognized | the longest redelivery upstream (`IDP-3`) |
| `lockTtl` | **milliseconds** | how long a crashed process blocks others | one execution, plus margin |
| `waitTimeout` | **milliseconds** | how long a concurrent caller waits in line | the caller's remaining budget (`14-resilience.md` › `RSL-2`) |

```text
❌  ttl: 3_600            an hour, behind a queue that dead-letters after a day
✅  ttl: 259_200          three days, outliving the queue's own policy

❌  lockTtl: 86_400_000   a crashed process blocks the key for a day
✅  lockTtl: 30_000       long enough for one execution to finish
```

### Shape

```typescript
@Idempotent((args: [ProcessPaymentInput]) => args[0].messageId, {
  mode: 'skip',
  ttl: 259_200,
})
async execute(input: ProcessPaymentInput) {
  return this.chargeOrderUseCase.execute(input)
}
```

### ❌ Never do

```typescript
// ❌ [IDP-1] the clock as a key — two runs of the same window are two keys
@Idempotent(() => `payout:${Date.now()}`)

// ❌ [IDP-1, ARC-TOP-7] a scheduled run keyed on "now" instead of on its window
async execute() { await this.settle(new Date()) }

// ❌ [IDP-2] a queue handler storing a result nobody will ever read back
@Idempotent(resolver, { mode: 'replay' }) // no caller is waiting

// ❌ [IDP-3] a key that expires before the delivery that repeats it
@Idempotent(resolver, { ttl: 60 })

// ❌ [IDP-4] weakening the fingerprint so a mismatched retry "just works"
@Idempotent((args) => args[0].key, { /* then ignoring the payload difference */ })

// ❌ [ARC-IDM-6] making the operation idempotent by checking first, then writing
const existing = await this.repo.findByKey(key) // two callers both find nothing
if (!existing) { await this.repo.create(input) } // and both create

// ❌ [ARC-DEL-5] catching the failure so the message is never redelivered —
//    the key is burned and the work never happened
try { await this.charge(input) } catch { return }
```
