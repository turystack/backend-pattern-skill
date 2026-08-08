# Error Handling

**Concept.** A business error is typed and mapped to a stable API status/code; never a loose string, never a leaking stack. A central catalogue keeps codes and messages consistent across domains.

> **How to read this file.** Two parts, deliberately separated:
> - **🌐 Generic pattern** — the **portable law**. Holds in any stack (NestJS, PHP, etc.); the text stays true even after swapping frameworks. This is what review enforces as an invariant.
> - **🛠️ Project-specific** — the **code** that implements each rule in the current stack (TypeScript · NestJS · @turystack · Drizzle · Zod). Swapping stacks rewrites **only** this part; the law above does not change.
>
> Each rule carries an id `ERR-n` linking the law (generic) to the code (specific). Gates bind by **id**, not by file/line. `ERR-L*` rules are **stack lints**: they exist only because of the current language's ergonomics (they invert in another stack), but they stay id-registered and enforced in this project.

---

## 🌐 Generic pattern (portable — stack-independent)

**ERR-1 — single central catalogue.** There is one error catalogue per project, not one file per module. Keys in `snake_case`. **[ERR-1]**

**ERR-2 — automatic `not_found`.** Every module gets `not_found` automatically; never declare it by hand in the module's key list. **[ERR-2]**

**ERR-3 — category class + catalogue key.** Every business exception is thrown with: (a) the typed class matching the error's **category** and (b) the catalogue key — never a string literal. **[ERR-3]**

**ERR-4 — category → layer mapping.** The layer that throws each error category is fixed by the architecture: **[ERR-4]**

- **invalid input (400)** → edge/route (schema validation).
- **not found (404)** → use-case (FK/PK existence check).
- **business conflict (409)** → entity guard.
- **not authenticated (401) / no permission (403)** → IAM layer.

> The order is not arbitrary: you cannot evaluate a business guard on a record whose existence you have not proven. The category→concrete class mapping is project-specific.

### Invariants (the law the gates enforce)

> Bind by **id**. `constitutional` = portable invariant (holds in any stack). `stack lint` = exists only because of the current language's ergonomics and inverts in another stack — still id-registered and enforced here. The **detector** (how the gate catches the violation) is project-specific and lives in the 🛠️ part below.

| ID | Law (one line) | Class | Detector (🛠️) |
|---|---|---|---|
| ERR-1 | One catalogue per project; `snake_case` keys; never an exceptions file per module | constitutional | Scenario 1 / ❌ |
| ERR-2 | `not_found` automatic per module; never declared by hand in the key array | constitutional | Scenario 1 / ❌ |
| ERR-3 | Throw with category class + catalogue key; never a string literal | constitutional | Scenarios 1–2 / ❌ |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-ERR-1` | One catalogue per product. |
| `ARC-ERR-2` | The code is the contract; the message is human. |
| `ARC-ERR-3` | Errors thrown with class and key. |
| `ARC-ERR-4` | The category has a layer. |
| `ARC-ERR-5` | Internal detail never reaches the client. |
| `ARC-ERR-8` | A remote read has five outcomes; all of them decided. |

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

- **ERR-1** — a single `src/exceptions.ts` with `createExceptions` from `@turystack/exceptions`; never `src/domains/x/x.exceptions.ts`. The same file declares `export type Exceptions = InferExceptionCodes<typeof exceptions>` — the union of **every** code the API can return. There is no translation dictionary: the error body comes out of the global `AppErrorTransform` of `Server.create` as `{ statusCode, code, message, ...metadata }`, with `code` = the stable catalogue code (the client translates by code, if it wants to). In OpenAPI, every error response references the model named **`Exception`** in `components.schemas` (registered by `Server.create`) — the generated SDK gets a single `Exception` type.
- **ERR-2** — `e.module('order', { conflict: ['already_paid'] })` — `createExceptions` injects `notFound` (code `order.not_found`) into every module; it never shows up in the groups.
- **ERR-3** — the builder generates **one class per code**, grouped by **HTTP semantics** (`conflict`, `unprocessableEntity`…): `throw new exceptions.order.alreadyPaid({ orderId })`. Each class carries a stable `code` (`order.already_paid`), a status and `metadata` — never `throw new Error('string')` nor a loose message.
- **ERR-4** — 400 → Zod on the route; 404 → use-case (the repositories from `@turystack/nestjs-database` already throw `RecordNotFoundError`/`RecordNotCreatedError`, codes `record_not_found`/`record_not_created`); 409 → entity guard; 401/403 → `IamUnauthorizedException`/`IamForbiddenException` from `@turystack/nestjs-iam`. Table of the groups below; ★ marks the ones on the validation ladder.

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

**Scenario 1 — central catalogue with `createExceptions`:** `[ERR-1, ERR-2, ERR-3]`
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

**Scenario 2 — throw the typed class from the catalogue:** `[ERR-3]`
```typescript
// use inside a use-case/entity — camelCase accessor, stable snake_case code
import { exceptions } from '@/exceptions'

throw new exceptions.order.notFound({ orderId })      // auto — code order.not_found
throw new exceptions.order.alreadyPaid({ orderId })   // code order.already_paid
```

**Scenario 3 — the API's `Exceptions` type + a route documenting the classes it can return:** `[ERR-1, ERR-3]`
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
// ❌ [ERR-3] loose string / generic class instead of the typed catalogue class
throw new Error('Order not found')

// ❌ [ERR-1] one exception file per module (must be a central catalogue)
// src/domains/order/order.exceptions.ts

// ❌ [ERR-2] declaring not_found by hand (it is automatic)
order: e.module('order', { notFound: ['not_found'], conflict: ['already_paid'] })
```
