# Controllers

**Concept.** Transport boundary: it translates HTTP ↔ application call. Thin — it validates the input, delegates to the use-case, returns the response. No business rule, no data access, no transaction.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> @turystack · Zod. The split, `XXX-n` versus `XXX-Ln`, and why an `ARC-…` law
> is cited and never restated: `turystack-backend-pattern` › *How a section is
> written*.

---

**Rules defined here:** `CTL-2` · `CTL-4` · `CTL-5` · `CTL-6` · `CTL-7` ·
`CTL-8` · `CTL-L1` · `CTL-L2` — the law is the *Invariants* table below; every
❌ item cites the id it violates.

**Retired ids:** `CTL-1` · `CTL-3` — retired, not renumbered. A review or
commit citing one points at a rule that no longer exists; the number is never
reused.

## 🌐 Generic pattern (portable — stack-independent)

**ARC-DEL-1 — thin controller.**

The controller validates the input shape at the edge, delegates execution to the use-case (typically 1 route → 1 use-case) and returns the response. It never contains a business rule, never accesses a repository or adapter directly, never runs a transaction. **[ARC-DEL-1]**

**CTL-2 — multiple surfaces.**

A codebase may expose several **HTTP surfaces**, one per consumer (e.g. customer app, operations console, partner app, backoffice), each with its own route prefix and its own OpenAPI document. Webhooks from external providers live in an isolated surface, separate from the consumption APIs. The logic lives in the shared domain layer; the controller is only the per-surface edge. **[CTL-2]**

**ARC-SEC-1 — scope from the auth context, never from client input.**

On every surface with an authenticated user, the scope identifier (`organizationId`, `userId`) is extracted from the authenticated context and injected by the controller — **never** trusted from the client's body/query/params:

- **Client surface** → scope **forced** from the authenticated context (the identity itself), never from input.
- **Operator surface** → the **operator organization's** scope, forced by the permission; limited to their org.
- **Internal surface** → **no** forced scope; the organization identifier (and the like) comes in as an **optional query** in the standard input, since the internal user queries **across** organizations.

The use-case receives the id and asserts/filters internally. **[ARC-SEC-1]**

**CTL-4 — permission gating (mandatory on every protected route).**

Every route that requires authenticated access must combine two levels of protection:

1. **Surface access** — checks whether the profile has permission to access the API (e.g. `access:bko`, `access:ops`). Applies to **all** endpoints of the surface.
2. **Resource permission** — checks whether the profile has permission for the specific resource (e.g. `invite:read`) **and** restricts the scope (see ARC-SEC-1). Applies per route.

Fully protected controllers stack both; controllers with a public/protected mix omit the surface level from the class and apply it per route. **[CTL-4]**

**CTL-5 — endpoint ordering by HTTP method.**

Inside the controller, endpoints are always listed in this order: GET → POST → PUT → PATCH → DELETE. **[CTL-5]**

**CTL-6 — terse route documentation.**

Each route's `summary`/`description` are terse: action + noun, with no scope qualifiers ("of the organization", "for operators", "in the global catalogue"). **[CTL-6]**

**CTL-7 — business action = custom method with `:verb` (ALWAYS, MANDATORY).**

A route that performs an **action** on a resource (not plain CRUD) uses the custom method pattern: the verb goes into the path **separated by `:`**, attached to the resource — `POST /invoices/:invoiceId:pay`, `POST /orders/:orderId:cancel`, `POST /users/:userId:activate`. Never the verb as a segment (`/invoices/:invoiceId/pay`): a path segment is a **resource/relationship** (`/workspaces/:workspaceId/invites`), never an action. The custom method's HTTP method is always `POST`. **[CTL-7]**

**CTL-8 — a uniqueness rule exposes its own availability read.**

A consumer that needs to know whether a value is still free gets a **dedicated
endpoint**. A filtered listing answered with one row is not an availability
check: it is a **second implementation of the constraint**, written by whoever
consumes it, and it diverges the first time the rule gains a condition —
case-insensitive, scoped per organization, ignoring archived rows. The authority
owns the rule; the consumer receives the verdict. If the endpoint does not exist
yet, that is a contract blocker (`ARC-CTR-5`), never a license to reconstruct
the rule in the consumer.

The shape is standard so it is never negotiated per feature:

```text
path         GET /{resources}/availability
query        field · value · exclude{Entity}Id?
status       200 — always
body         { available, field, value, reason? }
reason       { code, message } — the SAME catalogue code the write throws
permission   the permission of the write it guards, not the resource's read
```

Five properties carry the whole design:

**`availability` is a noun, so it is a sub-resource** of the collection and
`CTL-7` stays intact. It is not a custom method: `::` is POST-only by
construction, and nothing here is written.

**`field` is an enum of the entity's unique fields**, so the generated contract
publishes the list of what is actually unique — a consumer probing a field that
carries no constraint fails to compile instead of asking a question the backend
cannot answer.

**Taken is not an HTTP error.** The answer is `200` with `available: false`. A
`409` would land the consumer in its error branch, where a taken value and a
dead network look identical — and the consumer would have to guess which of the
two should block the submit.

**The reason carries the same catalogue code the write throws.** The read that
warns and the write that refuses speak with one voice (`ARC-ERR-1`,
`ARC-ERR-2`), so the sentence the user reads before submitting and the one they
read after a lost race are the same sentence, and neither can drift from the
other.

**The read is advisory.** It reserves nothing and locks nothing: between the
answer and the write, another request may take the value. The constraint at the
authority is what decides (`ARC-CON-4`) — same non-binding property a blast
radius preview has (`ARC-CON-11`). `exclude{Entity}Id` exists because an edit
must not collide with its own record.

Two narrow adjustments, each with its reason:

- when the probed value is **PII** (an email, a document, a phone), the read is
  `POST /{resources}/availability` with the value in the body — a `GET` would
  put it in a URL, and a URL reaches logs, referrers and proxies (`ARC-SEC-7`).
  This is the only sanctioned non-writing `POST`;
- when the read is **public** (a sign-up screen checking a free username), it is
  an enumeration oracle and carries a rate limit like any other abuse surface
  (see 11-security.md). **[CTL-8]**

### Invariants (the law the gates enforce)


| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| CTL-2 | Multiple surfaces with their own prefix/doc; webhooks in an isolated surface | constitutional | `manual` | Scenarios 1–4 |
| CTL-4 | Every protected route stacks surface-access + resource-permission with scope | constitutional | `gate:acl-coverage` | Scenarios 1–2, 4 / ❌ |
| CTL-5 | Endpoints ordered: GET → POST → PUT → PATCH → DELETE | constitutional | `gate:route-order` | Scenario 3 / (see 00-overview) |
| CTL-6 | Terse summary/description: action + noun; no scope qualifiers | constitutional | `manual` | Scenarios 1–4 / ❌ |
| CTL-7 | Business action = custom method `:verb` in the path (`/invoices/:invoiceId:pay`), always POST; the verb never becomes a segment | constitutional | `gate:route-shape` | Mechanism / ❌ |
| CTL-8 | A uniqueness rule is answered by `GET /{resources}/availability` (`field`/`value`/`exclude{Entity}Id?`), always `200`, with the same catalogue code the write throws and the permission of the write it guards; a filtered listing is never an availability check | constitutional | `manual` | Scenario 5 / ❌ |
| CTL-L1 | Main schemas registered via `schemas:{}` on the controller decorator (reusable model in the docs) | stack lint | `gate:route-schemas` | Scenario 1 |
| CTL-L2 | DTOs derive from canonical schemas via `.pick`/`.extend`; only `*Request`/`*SchemaResponse` exported | stack lint | `grit:dto-from-schema` | Scenario 4 |

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-DEL-1` | The delivery boundary is thin. | the controller validates shape, injects scope and calls one use-case |
| `ARC-SEC-1` | Scope comes from the authenticated context. | `@AuthenticatedProfile()`, never `@Body`/`@Query` for scope |
| `ARC-SEC-3` | Authorization is operation plus resource. | `@ACL('resource:action', …)` with the resource context declared by the route |
| `ARC-ERR-2` | The code is stable and is the contract. | the availability read answers with the same catalogue code its write throws (`CTL-8`) |
| `ARC-CON-4` | Uniqueness is enforced by a constraint, not by a read. | the availability read is advisory and reserves nothing; the constraint decides at write time (`CTL-8`) |


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into the code**: the standard is zero comments (see Code Quality).

**Mechanisms per rule (NestJS · @turystack):**

- **ARC-DEL-1** — `@Inject(CreateOrderUseCase)`; the route handler calls `useCase.execute(input)` (typically 1 route → 1 use-case); the controller never injects a repository, `DatabaseService` or direct adapters. There is never a business-rule `if`/`throw` nor `@Transactional` in the controller.
- **CTL-2** — `@Controller({ path, prefix, tag, schemas? })` from `@turystack/nestjs-server`; `prefix` defines the **project** (`app`, `admin`, `internal`). Each project goes into the `projects` array of `Server.create` and gets its own OpenAPI document + Scalar reference (`/{globalPrefix}/v1/{prefix}/openapi` · `/reference`). Single-project API: no `prefix`, controllers in `src/controllers/{domain}/`; multi-project: `src/controllers/{project}/{domain}/`.
- **ARC-SEC-1** — `@AuthenticatedProfile() profile: IamProfile` (from `@turystack/nestjs-iam`) extracts the authenticated scope; the client surface injects `profile.userId`/`profile.organizationId` explicitly into the use-case input; a route scoped to a workspace declares the context in `@ACL('resource:action', (request) => ({ workspaceId: request.params.workspaceId }))` — IAM cross-checks the declared context against the conditions of the profile's role; the internal surface receives `organizationId` as an optional field in the query schema.
- **CTL-4** — `@Auth()` at the class level (fully protected controller) **and** `@ACL('resource:action', contextCallback?)` per route — `@ACL` already stacks authentication + permission check. In a controller with a public/protected mix, omit `@Auth` from the class and apply it per route.
- **CTL-5** — ordering convention (see `00-overview`).
- **CTL-6** — `@Route({ summary: 'Create Order', description: 'Creates an order.' })`; never "Creates an order in the organization catalogue."
- **CTL-7** — turystack syntax in `@Route`: the verb comes in with **`::`** — `@Route({ method: 'POST', path: ':invoiceId::pay', ... })` matches `POST /invoices/inv-123:pay` and extracts `invoiceId`; collection custom methods work the same (`'reports::generate'`). The decorator internally converts it to the escape the Express 5 path matcher requires (`\:` — the documented NestJS 11 pattern for reserved characters) and **validates at boot**: `::` with a method other than POST throws `[Route] custom method path '...' requires method POST`. No escape ever appears in project code.
- **CTL-8** — `@Route({ method: 'GET', path: 'availability', summary: 'Get Product Availability' })`, first in the controller under `CTL-5`. The request schema declares `field: z.enum(...)` built from the entity's unique keys and `exclude{Entity}Id` optional (`CTL-L2`); the query use-case is `Get{Entity}AvailabilityUseCase` (`UC-1` — a query is a use-case too) and the repository method stays a data verb, `existsByUniqueField`/`findOneByName` (`REP-1`). The reason is built from **the same catalogue class the write throws** — `new exceptions.product.nameAlreadyExists({ name })` read for its `code`/`message` instead of being thrown — which is what makes drift between the two impossible. The permission is the write's: `@ACL('product:create')` on the create surface, `@ACL('product:update')` where it guards an edit. PII in the probed value moves the route to `POST` with the value in the body; a public surface adds the rate limit guard from `@turystack/nestjs-rate-limit` (see 11-security.md).
- **CTL-L1** — `@Controller({ schemas: { Order: { schema: orderResponse }, OrderStatus: { schema: orderStatusSchema }, OrderIdentity: { schema: orderIdentitySchema } } })` — registers the domain's main schemas as named models in the OpenAPI document; never anonymous inline.
- **CTL-L2** — `createRequestSchema({ body: orderSchema.pick({...}) })` / `.extend({...})` in the `{domain}.schemas.ts` next to the controller; exports only `*Request` / `*SchemaResponse` (the name must contain `Schema`); bodies inlined and derived from the domain schemas. A field the **user types** is declared with `@turystack/fields` rather than picked as-is — the domain schema types what a row holds, and a row never holds `'   '` (`SCH-9`, `SCH-L5`).

### ✅ How to do it

**Scenario 1 — protected route, injects the use-case, terse summary, schemas in OpenAPI:** `[ARC-DEL-1, CTL-4, CTL-6, CTL-L1]`
```typescript
@Controller({
  path: 'invoices',
  prefix: 'admin',
  tag: 'Invoice',
  schemas: {
    Invoice: { schema: invoiceResponse },
    InvoiceStatus: { schema: invoiceStatusSchema },
    InvoiceIdentity: { schema: invoiceIdentitySchema },
  },
})
@Auth()
export class InvoiceAdminController {
  constructor(@Inject(CreateInvoiceUseCase) private readonly createInvoiceUseCase: CreateInvoiceUseCase) {}

  @Route({ method: 'POST', path: '', summary: 'Create Invoice', description: 'Creates an invoice.' })
  async create(@Request(createInvoiceRequest) req: RequestInput<typeof createInvoiceRequest>) {
    return this.createInvoiceUseCase.execute(req.body)
  }
}
```

**Scenario 2 — resource authorization with the scope declared in the ACL:** `[ARC-SEC-1, CTL-4]`
```typescript
@Route({ method: 'GET', path: 'workspaces/:workspaceId/invites', summary: 'List Invites', description: 'Lists invites.' })
@ACL('invite:read', (request) => ({ workspaceId: request.params.workspaceId }))
async list(
  @AuthenticatedProfile() profile: IamProfile,
  @Request(listInvitesRequest) req: RequestInput<typeof listInvitesRequest>,
) {
  return this.listInvitesUseCase.execute({ ...req.query, organizationId: profile.organizationId })
}
```

**Scenario 3 — client API: scope injected from `@AuthenticatedProfile`, never from the input:** `[ARC-SEC-1, CTL-4, CTL-5]`
```typescript
@Controller({ path: 'orders', prefix: 'app', tag: 'Order' })
@Auth()
export class OrderAppController {
  constructor(
    @Inject(ListOrdersUseCase) private readonly listOrdersUseCase: ListOrdersUseCase,
    @Inject(CreateOrderUseCase) private readonly createOrderUseCase: CreateOrderUseCase,
  ) {}

  @Route({ method: 'GET', path: '', summary: 'List Orders', description: 'Lists orders.' })
  async list(
    @AuthenticatedProfile() profile: IamProfile,
    @Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>,
  ) {
    return this.listOrdersUseCase.execute({ ...req.query, userId: profile.userId })
  }

  @Route({ method: 'POST', path: '', summary: 'Create Order', description: 'Creates an order.' })
  async create(
    @AuthenticatedProfile() profile: IamProfile,
    @Request(createOrderRequest) req: RequestInput<typeof createOrderRequest>,
  ) {
    return this.createOrderUseCase.execute({
      ...req.body,
      organizationId: profile.organizationId,
    })
  }
}
```

**Scenario 4 — same resource, three projects (app / admin / internal):** `[CTL-2, ARC-SEC-1, CTL-4, CTL-L2]`
```typescript
// main.ts — each project in the projects array gets its own prefix + OpenAPI doc + Scalar reference
Server.create(AppModule, (config) => ({
  port: config.get('PORT'),
  title: 'Acme API',
  description: 'Acme backend',
  docs: { provider: 'scalar' },
  projects: [
    { name: 'app', title: 'App API', prefix: 'app' },
    { name: 'admin', title: 'Admin API', prefix: 'admin' },
    { name: 'internal', title: 'Internal API', prefix: 'internal' },
  ],
}))
```

```typescript
// App (client) — scope FORCED from @AuthenticatedProfile
@Controller({ path: 'orders', prefix: 'app', tag: 'Order' })
@Auth()
export class OrderAppController {
  @Route({ method: 'GET', path: '', summary: 'List Orders', description: 'Lists orders.' })
  async list(
    @AuthenticatedProfile() profile: IamProfile,
    @Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>,
  ) {
    return this.listOrdersUseCase.execute({ ...req.query, userId: profile.userId })
  }
}

// Admin (operator) — resource permission via @ACL; the operator organization's scope
@Controller({ path: 'orders', prefix: 'admin', tag: 'Order' })
@Auth()
export class OrderAdminController {
  @Route({ method: 'GET', path: '', summary: 'List Orders', description: 'Lists orders.' })
  @ACL('order:read')
  async list(
    @AuthenticatedProfile() profile: IamProfile,
    @Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>,
  ) {
    return this.listOrdersUseCase.execute({
      ...req.query,
      organizationId: profile.organizationId,
    })
  }
}

// Internal — no forced scope; organizationId is an OPTIONAL query in the standard input
@Controller({ path: 'orders', prefix: 'internal', tag: 'Order' })
@Auth()
export class OrderInternalController {
  @Route({ method: 'GET', path: '', summary: 'List Orders', description: 'Lists orders.' })
  async list(
    @Request(listOrdersInternalRequest) req: RequestInput<typeof listOrdersInternalRequest>,
  ) {
    return this.listOrdersUseCase.execute(req.query)
  }
}
```

```typescript
// internal: organizationId comes in as an OPTIONAL query, in the {domain}.schemas.ts next to the controller
import { PagePaginationSchema } from '@turystack/query-dsl'

export const listOrdersInternalRequest = createRequestSchema({
  // SCH-L3: pagination comes from the package, not from two coerced numbers —
  // it is the same `page`/`limit` the SDK, the table and the response meta use
  query: PagePaginationSchema.extend({
    organizationId: orderSchema.shape.organizationId.optional(),
    status: orderStatusSchema.optional(),
  }),
})
```

**Scenario 5 — availability read for a uniqueness rule:** `[CTL-8, CTL-5, CTL-4, CTL-L2, ARC-SEC-1]`
```typescript
// product.schemas.ts — field is the enum of what is ACTUALLY unique; the consumer
// cannot probe a field with no constraint behind it
import { RequiredStringSchema } from '@turystack/fields'

export const productUniqueFieldSchema = z.enum(['name', 'sku'])

export const getProductAvailabilityRequest = createRequestSchema({
  query: z.object({
    field: productUniqueFieldSchema,
    value: RequiredStringSchema(), // SCH-L5: z.string().min(1) probes availability for '   '
    // without this, every edit form collides with its own record
    excludeProductId: productSchema.shape.productId.optional(),
  }),
})

export const productAvailabilitySchemaResponse = z.object({
  available: z.boolean(),
  field: productUniqueFieldSchema,
  value: z.string(),
  reason: z
    .object({ code: z.string(), message: z.string() })
    .optional(),
})
```

```typescript
// product-app.controller.ts — availability is a GET, so it leads the controller (CTL-5)
@Route({
  method: 'GET',
  path: 'availability',
  summary: 'Get Product Availability',
  description: 'Checks whether a unique product value is available.',
})
// the permission of the WRITE it guards, never 'product:read' (CTL-8)
@ACL('product:create')
async availability(
  @AuthenticatedProfile() profile: IamProfile,
  @Request(getProductAvailabilityRequest) req: RequestInput<typeof getProductAvailabilityRequest>,
) {
  return this.getProductAvailabilityUseCase.execute({
    ...req.query,
    organizationId: profile.organizationId,
  })
}
```

```typescript
// exceptions.ts — one catalogue entry per unique field, declared once (ARC-ERR-1, ERR-L1)
e.module('product', { conflict: ['name_already_exists', 'sku_already_exists'] })
```

```typescript
// get-product-availability.use-case.ts — a query is a use-case too (UC-1)
const CONFLICT_BY_FIELD = {
  name: exceptions.product.nameAlreadyExists,
  sku: exceptions.product.skuAlreadyExists,
}

export class GetProductAvailabilityUseCase {
  async execute(input: GetProductAvailabilityInput) {
    const taken = await this.productRepository.existsByUniqueField(input)

    if (!taken) {
      return { available: true, field: input.field, value: input.value }
    }

    // the same catalogue class the create/update throws — read, not thrown.
    // one source for the pair means the warning and the refusal cannot drift (ARC-ERR-2)
    const conflict = CONFLICT_BY_FIELD[input.field]({ [input.field]: input.value })

    return {
      available: false,
      field: input.field,
      value: input.value,
      reason: { code: conflict.code, message: conflict.message },
    }
  }
}
```

> Taken answers `200` with `available: false` — never a `409`. In the consumer, an
> error status is indistinguishable from a dead network, and the frontend has to
> tell the two apart to decide whether the submit is blocked
> (`turystack-frontend-pattern` › `COM-10`).

## Transport contracts

- The domain's main schema is canonical. Requests derive from it with
  `pick`/`omit`/`extend`, without redeclaring fields.
- The request schema sits next to the controller because it belongs to the HTTP boundary.
- Export only contracts that another boundary reuses.
- The response represents an entity or an explicit public schema; it never leaks
  a database row, a secret or an internal field.
- Route params, query and body are validated before the use-case.
- An incompatible contract change requires a new version or a transition strategy;
  silently renaming a field is not allowed.

The concrete request/response helpers and the OpenAPI generation belong to the
`@turystack/nestjs-server` documentation.

### ❌ Never do

```typescript
// ❌ [ARC-DEL-1] injecting a repository/DatabaseService/adapter into the controller
constructor(@Inject(OrderRepository) private readonly orderRepository: OrderRepository) {}

// ❌ [ARC-DEL-1] business rule / transaction in the controller
async create(@Request(s) req) {
  if (await this.orderRepository.existsByCode(req.body.code)) throw new ConflictException(...)
}

// ❌ [CTL-6] summary with a scope qualifier
description: 'Lists invites of the organization for operators.'  // use 'Lists invites.'

// ❌ [CTL-7] action as a path segment — a segment is a resource, never a verb
@Route({ method: 'POST', path: ':invoiceId/pay', ... })      // use ':invoiceId::pay'

// ❌ [CTL-7] custom method outside POST — the decorator throws at boot
@Route({ method: 'GET', path: ':invoiceId::pay', ... })

// ❌ [CTL-7] business action disguised as CRUD instead of a custom method
@Route({ method: 'PATCH', path: ':orderId', ... })           // body { status: 'canceled' } → use ':orderId::cancel'

// ❌ [ARC-SEC-1] in a client API, trusting the scope coming from the client (forgeable)
async list(@Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>) {
  return this.listOrdersUseCase.execute({ userId: req.query.userId }) // userId came from the client!
}

// ❌ [ARC-SEC-1] accepting organizationId/userId in the DTO of a client API
body: orderSchema.pick({ items: true, organizationId: true })

// ❌ [CTL-8] no availability endpoint, so the consumer reconstructs the rule from a listing —
// the constraint now has two implementations, and the copy is the one that diverges
// consumer: useListProducts({ name }) → data.length > 0

// ❌ [CTL-8] taken answered as an HTTP error — the consumer cannot tell it from a dead network
async availability(@Request(s) req) {
  if (await this.productRepository.existsByUniqueField(req.query)) {
    throw new exceptions.product.nameAlreadyExists({ name: req.query.value })
  }
}

// ❌ [CTL-7, CTL-8] verb in the path — availability is a noun, a sub-resource of the collection
@Route({ method: 'GET', path: 'check-name', ... })          // use path: 'availability'
@Route({ method: 'POST', path: '::check-name', ... })       // custom methods are for actions

// ❌ [CTL-8] one endpoint per unique field — the rule is one concept, the enum carries the field
@Route({ method: 'GET', path: 'name-availability', ... })
@Route({ method: 'GET', path: 'sku-availability', ... })

// ❌ [CTL-8] availability gated by the resource's read permission — it guards a write
@ACL('product:read')                                        // use 'product:create' / 'product:update'

// ❌ [CTL-8, ARC-SEC-7] PII in the query string — a URL reaches logs, referrers and proxies
@Route({ method: 'GET', path: 'availability', ... })        // ?field=email&value=ana@acme.com
                                                            // use POST + value in the body

// ❌ [CTL-8] availability with no self-exclusion — the edit form collides with its own record
query: z.object({ field: productUniqueFieldSchema, value: z.string() })

// ❌ [CTL-8] hand-written reason — it drifts from the message the write throws
reason: { code: 'name_taken', message: 'This name is already in use.' }

// ❌ [SCH-L3] pagination hand-rolled in the query schema — and under a name the
// rest of the stack does not use, so the SDK, the table and the response meta
// each carry a different one
page: z.coerce.number().int().min(1).default(1)
pageSize: z.coerce.number().int().min(1).max(50).default(20)
// use PagePaginationSchema.extend({ ... }) from '@turystack/query-dsl'
```
