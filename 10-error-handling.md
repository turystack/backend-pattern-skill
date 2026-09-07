# Error Handling

**Concept.** A business error is typed and mapped to a stable API status/code; never a loose string, never a leaking stack. A central catalogue keeps codes and messages consistent across domains.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> @turystack · Drizzle · Zod. The split, `XXX-n` versus `XXX-Ln`, and why an
> `ARC-…` law is cited and never restated: `turystack-backend-pattern` › *How
> a section is written*. Almost every rule here is constitutional and cited as
> `ARC-ERR-n`: what an error catalogue owes its consumers does not change when
> the framework does. Only `ERR-L1` is local — a **stack lint**, existing
> because of what `@turystack/exceptions` provides, and enforced here all the
> same.

---

**Rules defined here:** `ERR-L1` — the law is the *Invariants* table below;
every ❌ item cites the id it violates.

## 🌐 Generic pattern (portable — stack-independent)

**The layer that throws each category is fixed** — this is `ARC-ERR-4` at the
granularity of the backend's HTTP statuses:

- **invalid input (400)** → edge/route (schema validation).
- **not found (404)** → use-case (FK/PK existence check).
- **business conflict (409)** → entity guard.
- **not authenticated (401) / no permission (403)** → IAM layer.

> The order is not arbitrary: you cannot evaluate a business guard on a record whose existence you have not proven. The category→concrete class mapping is project-specific.

### Invariants (the law the gates enforce)


| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| ERR-L1 | Catalogue keys in `snake_case`; `not_found` is provided per module by the library and never hand-declared in the key array | stack lint | `grit:no-blind-error-cast` | Scenario 1 / ❌ |

**Where the old local ids went.** `ERR-1`, `ERR-3` and `ERR-4` restated laws the
constitution already owned, which meant one law with two ids and a gate that
could bind to either. They map as `ERR-1`→`ARC-ERR-1`, `ERR-3`→`ARC-ERR-3`,
`ERR-4`→`ARC-ERR-4`. What was genuinely local — the key convention and the
library-provided `not_found` — survives as the stack lint `ERR-L1` (it was
`ERR-2`).

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-ERR-1` | A domain owns its codes and publishes them, under its own name. | `domains/<name>/src/support/<name>.exceptions.ts`, exported by that domain's barrel |
| `ARC-ERR-2` | The code is the contract; the message is human. | the client branches on `code`, never on the message |
| `ARC-ERR-3` | Thrown with a category class and a catalogue key, never a literal. | `throw new exceptions.order.notFound({ orderId })` |
| `ARC-ERR-4` | The category decides the layer. | the four-line map above |
| `ARC-ERR-5` | Internal detail never reaches the client. | the filter serializes code + message; no stack, no SQL |
| `ARC-ERR-6` | The consumer branches on the code, never on the message. | the published catalogue is what the frontend imports |
| `ARC-ERR-7` | Every error becomes feedback or a propagated failure. | no empty `catch`, no catch that only logs |
| `ARC-ERR-8` | A remote read has five outcomes; all of them decided. | applies when this backend consumes a third-party API |

`ARC-ERR-8` applies whenever this backend is a **consumer**: a call to a
third-party API, an adapter, an integration. An empty list is not a failure, a
paginated response is not the complete set, and a timeout is not "no results".
Treating the five as two is the same bug the frontend makes, under another name.


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Drizzle · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into the code**: the standard is zero comments (see Code Quality).

**Mechanisms per rule (NestJS):**

- **ARC-ERR-1** — `domains/<name>/src/support/<name>.exceptions.ts` with `createExceptions` from `@turystack/exceptions`, its module named after the domain, exported by that domain's barrel as `<name>Exceptions`. The same file declares `export type <Name>ExceptionCode = InferExceptionCodes<typeof exceptions>` — the union of **every** code the API can return. There is no translation dictionary: the error body comes out of the global `AppErrorTransform` of `Server.create` as `{ statusCode, code, message, ...metadata }`, with `code` = the stable catalogue code (the client translates by code, if it wants to). In OpenAPI, every error response references the model named **`Exception`** in `components.schemas` (registered by `Server.create`) — the generated SDK gets a single `Exception` type.
- **ERR-L1** — `e.module('order', { conflict: ['already_paid'] })` — `createExceptions` injects `notFound` (code `order.not_found`) into every module; it never shows up in the groups.
- **ARC-ERR-3** — the builder generates **one class per code**, grouped by **HTTP semantics** (`conflict`, `unprocessableEntity`…): `throw new orderExceptions.alreadyPaid({ orderId })`. Each class carries a stable `code` (`order.already_paid`), a status and `metadata` — never `throw new Error('string')` nor a loose message.
- **ARC-ERR-4** — 400 → Zod on the route; 404 → use-case (the repositories from `@turystack/nestjs-database` already throw `RecordNotFoundError`/`RecordNotCreatedError`, codes `record_not_found`/`record_not_created`); 409 → entity guard; 401/403 → `IamUnauthorizedException`/`IamForbiddenException` from `@turystack/nestjs-iam`. Table of the groups below; ★ marks the ones on the validation ladder.

**Available groups (`createExceptions` · `@turystack/exceptions`) — the ★ ones belong to this architecture's validation ladder:**

| Status | Group (`e.module`) | Base class | Typical use |
|---|---|---|---|
| 400 | `badRequest` ★ | `BadRequestError` | malformed input — covered by Zod **on the route** |
| 401 | `unauthorized` ★ | `UnauthorizedError` | not authenticated / invalid credential |
| 403 | `forbidden` ★ | `ForbiddenError` | authenticated, no permission (ACL/scope) |
| 404 | `notFound` (automatic) ★ | `NotFoundError` | FK/PK existence check fails (use-case) |
| 405 | `methodNotAllowed` | `MethodNotAllowedError` | HTTP method not allowed on the route |
| 409 | `conflict` ★ | `ConflictError` | business invariant (entity guard) |
| 410 | `gone` | `GoneError` | resource permanently removed |
| 422 | `unprocessableEntity` | `UnprocessableEntityError` | invalid semantics (when 400 is not enough) |
| 429 | `tooManyRequests` | `TooManyRequestsError` | rate limit exceeded |
| 500 | `internalServerError` | `InternalServerError` | unexpected error (avoid throwing it explicitly) |
| 502 | `badGateway` | `BadGatewayError` | external provider answered invalid |
| 503 | `serviceUnavailable` | `ServiceUnavailableError` | dependency unavailable |
| 504 | `gatewayTimeout` | `GatewayTimeoutError` | external provider timeout |

> All of them extend `AppError` (`statusCode` + stable `code` + `metadata`); `isAppError(error)` does the narrowing. `ValidationError` (400) and `ConcurrentUpdateError` (409) exist as standalone classes for use outside the catalogue.

### ✅ How to do it

**Scenario 1 — central catalogue with `createExceptions`:** `[ARC-ERR-1, ERR-L1, ARC-ERR-3]`
```typescript
// src/exceptions.ts — central catalogue; snake_case codes grouped by HTTP semantics
import { createExceptions } from '@turystack/exceptions'

export const exceptions = createExceptions((e) => ({
  order: e.module('order', {
    conflict: ['already_paid'],
    unprocessableEntity: ['empty_cart'],
  }),
}))
```

**Scenario 2 — throw the typed class from the catalogue:** `[ARC-ERR-3]`
```typescript
// use inside a use-case/entity — camelCase accessor, stable snake_case code
import { exceptions } from '@/exceptions'

throw new exceptions.order.notFound({ orderId })      // auto — code order.not_found
throw new exceptions.order.alreadyPaid({ orderId })   // code order.already_paid
```

**Scenario 3 — the API's `Exceptions` type + a route documenting the classes it can return:** `[ARC-ERR-1, ARC-ERR-3]`
```typescript
// src/exceptions.ts — next to the catalogue: the union of every possible code
import { createExceptions, type InferExceptionCodes } from '@turystack/exceptions'

export const exceptions = createExceptions((e) => ({ ... }))
export type Exceptions = InferExceptionCodes<typeof exceptions>
// 'order.not_found' | 'order.already_paid' | 'order.empty_cart' | ...

// on the route: declare the catalogue CLASSES — @Route derives status, code and
// OpenAPI examples from the class itself (the server only shows what the lib already generates)
@Route({
  method: 'POST',
  path: ':orderId::pay',
  summary: 'Pay Order',
  description: 'Pays an order.',
  responses: {
    200: orderSchemaResponse,
    exceptions: [exceptions.order.notFound, exceptions.order.alreadyPaid],
  },
})
```

### ❌ Never do

```typescript
// ❌ [ARC-ERR-3] loose string / generic class instead of the typed catalogue class
throw new Error('Order not found')

// ❌ [ARC-ERR-1] one exception file per module (must be a central catalogue)
// src/domains/order/order.exceptions.ts

// ❌ [ERR-L1] declaring not_found by hand (it is automatic)
order: e.module('order', { notFound: ['not_found'], conflict: ['already_paid'] })
```
