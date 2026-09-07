# Schemas

**Concept.** The schema is the **source of truth** for the domain's types. It mirrors the data model, and **every** type derives from it: the entity's fields, the repository's inputs and the controller's DTOs. It is the base of the stack — `Schema → Entity → Repository → UseCase → Controller`.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> @turystack · Zod. The split, `XXX-n` versus `XXX-Ln`, and why an `ARC-…` law
> is cited and never restated: `turystack-backend-pattern` › *How a section is
> written*.

---

**Rules defined here:** `SCH-1` · `SCH-2` · `SCH-3` · `SCH-4` · `SCH-5` ·
`SCH-8` · `SCH-9` · `SCH-10` · `SCH-L1` · `SCH-L2` · `SCH-L3` · `SCH-L4` ·
`SCH-L5` — the law is the *Invariants* table below; every ❌ item cites the id
it violates.

**Retired ids:** `SCH-6` · `SCH-7` — retired, not renumbered. A review or
commit citing one points at a rule that no longer exists; the number is never
reused.

## 🌐 Generic pattern (portable — stack-independent)

**SCH-1 — source of truth: types are derived, never hand-written.**

The schema is the only authoritative definition of the domain's fields and types. Every downstream type — entity fields, repository inputs, controller DTOs — derives from the schema by inference or projection. Never hand-write a type that duplicates what the schema already defines. **[SCH-1]**

**SCH-2 — file separation: schemas vs. types.**

Two files per feature, with exclusive responsibilities:

- `{feature}.schema.ts` — declares the schemas. Exports **only schemas**.
- `{feature}.types.ts` — exports **only types** derived from the schemas. Never hand-written types.

Merging the two creates circular dependencies and makes it harder to track what is a schema vs. a type. **[SCH-2]**

**SCH-3 — full coverage of the model's fields.**

The schema covers **every** field of the data model: PK, FKs, business fields, `createdAt`/`updatedAt`/`deletedAt`, and audit fields. Omitting fields creates a discrepancy between the schema and the database — the stack is left with no source of truth for the missing fields. **[SCH-3]**

**SCH-4 — every field carries machine-readable metadata.**

Every schema field carries inline metadata (`description` + `example` where applicable) — no exceptions. **Terse** descriptions: never "Timestamp when the product was created", "of the user", "in the global catalogue". The metadata feeds the auto-generated documentation; without it the API has no docs. **[SCH-4]**

**SCH-5 — enums and complex types extracted as named schemas.**

An enum or complex type is extracted as its own schema **before** the main schema. Never inline. The extracted schema gets the same metadata treatment. Reason: an enum schema is reusable (validation, coercion, docs), and extracting it first keeps the main schema clean. **[SCH-5]**

**ARC-CTR-2 — relationship imported, never duplicated.**

When the schema needs to represent a relationship with another domain, it **imports** that domain's schema — it never recreates the shape inline. Duplication creates silent divergence when the related domain changes. **[ARC-CTR-2]**

**ARC-CTR-3 — Identity: lean schema for cross-domain FK joins.**

Each domain exposes an **Identity** — a lean projection of the main schema containing only the PK + key fields (`name`/`email`/`document`). The Identity:

- is declared in the domain's own `{feature}.schema.ts` (like the extracted enums);
- has a derived type in `{feature}.types.ts`;
- is the only shape used when the domain appears as an **FK join** in another entity — never the full entity (heavy/recursive join).

An FK (`organizationId`) always carries the `organization: OrganizationIdentity` field alongside it. **[ARC-CTR-3]**

**SCH-8 — field ordering.**

Inside the main schema, always in this order: **PK → FKs (+ the join's Identity field alongside) → important fields → less important fields → booleans → status → timestamps + audit**. The order mirrors the entity and the data model. **[SCH-8]**

**SCH-9 — an input field validates what the user can actually type.**

A field that arrives from a human — a request body or a form — is validated against the shape **and** against the ways that shape is wrong in practice: whitespace-only text, a blank numeric field, an amount with too many decimals, a date the calendar does not have, a document whose check digits do not close. A type check that accepts `'   '` as a name is not validation; it is a cast.

The rules are declared once and shared by every surface that accepts the field, so the API and the form agree on what is valid. **[SCH-9]**

**SCH-10 — a validation failure carries a stable code, never a message to match on.**

Every failure exposes a machine-readable code that does not change with the validation library's version or locale. Callers branch on the code; the message is resolved from it. Matching on the text of an error is a contract nobody declared and everybody breaks. **[SCH-10]**

### Invariants (the law the gates enforce)


| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| SCH-1 | Schema is the source of truth; downstream types derived by inference/projection, never hand-written | constitutional | `manual` | Scenarios 1–2 / ❌ |
| SCH-2 | Schemas and derived types in separate files, each with an exclusive responsibility; never mixed | constitutional | `gate:file-roles` | Scenarios 1–2 / ❌ |
| SCH-3 | Schema covers every field of the model (PK, FKs, business, timestamps, audit) | constitutional | `manual` | Scenario 1 / ❌ |
| SCH-4 | Every field carries inline metadata (description + example); terse descriptions | constitutional | `gate:schema-metadata` | Scenarios 1–2 / ❌ |
| SCH-5 | Enum/complex type extracted as a named schema before the main schema; never inline | constitutional | `manual` | Scenarios 1–2 / ❌ |
| SCH-8 | Field ordering: PK → FKs (+ Identity) → important → booleans → status → timestamps + audit | constitutional | `manual` | Scenario 1 |
| SCH-9 | An input field validates the ways its shape is wrong in practice, not only the shape | constitutional | `manual` | Scenario 3 / ❌ |
| SCH-10 | A failure carries a stable code; callers branch on the code, never on the message | constitutional | `manual` | Scenario 3 / ❌ |
| SCH-L1 | Import from `zod`, never from `zod/v4` | stack lint | `biome:noRestrictedImports` | ❌ |
| SCH-L2 | Dates use `z.date()` (never `z.string()`); optional/nullable uses `.nullish()` | stack lint | `grit:no-string-date` | Scenario 1 / ❌ |
| SCH-L3 | Model schema is native Zod, no **local** wrappers; `z.infer` to derive types (never hand-write the shape) | stack lint | `grit:no-hand-written-model-type` | Scenarios 1–2 / ❌ |
| SCH-L4 | Metadata via `.meta({ description, example })` chained directly on the field | stack lint | `gate:schema-metadata` | Scenarios 1–2 / ❌ |
| SCH-L5 | Input fields come from `@turystack/fields`; hand-rolled `.trim().min(1)`, `z.coerce.number()` on a body and hand-written document checks banned | stack lint | `grit:no-hand-rolled-field` | Scenario 3 / ❌ |

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-CTR-1` | Contract declared once, types derive. | one Zod schema per concept; DTO and response derive with `.pick`/`.omit` |
| `ARC-CTR-2` | Relationship imported from the owning module. | import the other domain's schema; never redeclare its fields |
| `ARC-CTR-3` | Module exposes an Identity to be referenced. | the domain exports its Identity type for others to reference |
| `ARC-CTR-7` | Contract data stays derived; never copied into local state. | types inferred from the schema, never a parallel hand-written interface |

`ARC-CTR-1` kills the second **shape** — the hand-rewritten type. `ARC-CTR-7`
kills the second **copy** — the instance field that stores the result of a read
and goes stale in silence. `typecheck` catches the first; it catches nothing of
the second.


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into the code**: the standard is zero comments (see Code Quality).

**Mechanisms per rule (NestJS · @turystack):**

- **SCH-1** — `z.infer<typeof xxxSchema>` derives the type; `Pick`/`Omit`/`Partial` project it for downstream inputs (see `04-use-cases`, `05-repositories`).
- **SCH-2** — `{feature}.schema.ts` uses `export const xxxSchema = z.object(...)` + enums/Identity; `{feature}.types.ts` uses `import type { z } from 'zod'` + `export type Xxx = z.infer<typeof xxxSchema>`.
- **SCH-3** — the domain's Zod schema mirrors, field by field, the table defined in `src/database/database.schema.ts` (`defineDatabaseSchema` from `@turystack/nestjs-database`): PK (`z.uuid()`), FKs (`z.uuid()`), timestamps (`z.date()`), `deletedAt` (`z.date().nullish()`). Distinct roles, same model: the Zod schema validates the business shape; the database schema defines persistence (see `05-repositories`).
- **SCH-4** — `.meta({ description: '...', example: '...' })` on every field; description without "Timestamp when", "of the user", "in the global catalogue".
- **SCH-5** — `export const xxxStatusSchema = z.enum([...]).meta(...)` before `xxxSchema`; every enum and relationship sub-schema defined before the main one.
- **ARC-CTR-2** — `import { categoryIdentitySchema } from '@/domains/category/category.schema'`; never `z.object({ categoryId: z.string(), name: z.string() })` inline.
- **ARC-CTR-3** — `export const xxxIdentitySchema = xxxSchema.pick({ xxxId: true, name: true })` in the same `*.schema.ts`; `export type XxxIdentity = z.infer<typeof xxxIdentitySchema>` in `*.types.ts`.
- **SCH-8** — the field order in `z.object(...)` mirrors the convention: PK → FKs + Identity → important → booleans → status → timestamps.
- **SCH-L1** — the project uses Zod v4; the canonical import is `import { z } from 'zod'` — never `import { z } from 'zod/v4'`.
- **SCH-L2** — `z.date()` for `createdAt`/`updatedAt`/`deletedAt`; `.nullish()` for optional/nullable fields.
- **SCH-L3** — in the **model** schema use `z.object`, `z.string`, `z.uuid`, `z.boolean`, `z.array`, `z.enum` directly; never a wrapper invented in the domain file. A stack package is not a wrapper: `@turystack/fields` on input fields (`SCH-L5`) and `@turystack/query-dsl` on query parameters are the two exceptions, and both are mandatory where they apply.
- **SCH-L4** — `.meta(...)` chained directly on the field's schema (never in a comment nor in separate docs).
- **SCH-9** — the input schema (`{domain}.schemas.ts` next to the controller, `CTL-L2`) builds its user-facing fields with `@turystack/fields`: `RequiredStringSchema`, `EmailSchema`, `MoneySchema`, `DateOnlySchema`, `FileSchema`, `UrlSchema`, and `@turystack/fields/br` for `CpfSchema`/`CnpjSchema`/`PhoneSchema`. The model schema (Scenario 1) keeps native Zod — it mirrors the table, not a keyboard.
- **SCH-10** — every failure carries `params.code` from `FieldIssueCode`; `formatErrors(error, resolve)` flattens a `ZodError` into `Record<path, { code, message, params }>`. A missing value always resolves to `required`, never to `invalid_type`.
- **SCH-L5** — `import { RequiredStringSchema } from '@turystack/fields'`; never `z.string().trim().min(1)`, `z.coerce.number()` on a body field, `Math.round(value * 100)` for money, `z.coerce.date()` for a calendar date, or a hand-written CPF/CNPJ regex. Every one of those has a schema that already covers the case the hand-rolled version misses.

### ✅ How to do it

**Scenario 1 — main schema: extracted enum, Identity, FK join, full coverage, field ordering:** `[SCH-1, SCH-2, SCH-3, SCH-4, SCH-5, ARC-CTR-2, ARC-CTR-3, SCH-8, SCH-L1, SCH-L2, SCH-L3, SCH-L4]`

```typescript
import { z } from 'zod'

import { categoryIdentitySchema } from '@/domains/category/category.schema'

import { productPriceSchema } from './product-price.schema'

// SCH-5: enum extracted as a named schema before the main schema
export const productStatusSchema = z
  .enum(['ACTIVE', 'INACTIVE', 'DRAFT'])
  .meta({ description: 'Product status', example: 'ACTIVE' })

// SCH-3 + SCH-8: covers every field; order PK → FKs → important → booleans → status → timestamps
export const productSchema = z.object({
  productId: z.uuid().meta({ description: 'Product id' }),
  categoryId: z.uuid().meta({ description: 'Category id' }),
  category: categoryIdentitySchema, // ARC-CTR-2+ARC-CTR-3: FK join = imported Identity, never inline nor the full entity
  name: z.string().meta({ description: 'Product name', example: 'Headphones' }),
  prices: z.array(productPriceSchema), // main join (1:N) = full schema imported
  description: z.string().nullish().meta({ description: 'Product description' }), // SCH-L2: .nullish()
  isGift: z.boolean().meta({ description: 'Sold as gift', example: false }),
  status: productStatusSchema,
  createdAt: z.date().meta({ description: 'Creation timestamp' }), // SCH-L2: z.date()
  updatedAt: z.date().meta({ description: 'Last update timestamp' }),
  deletedAt: z.date().nullish().meta({ description: 'Deletion timestamp' }),
})

// ARC-CTR-3: the domain's own Identity — used when product becomes a join (FK) in another entity
export const productIdentitySchema = productSchema.pick({
  productId: true,
  name: true,
})
```

**Scenario 2 — types file: only derived types, never hand-written:** `[SCH-1, SCH-2, SCH-L3]`

```typescript
import type { z } from 'zod'

import type {
  productIdentitySchema,
  productSchema,
  productStatusSchema,
} from './product.schema'

export type Product = z.infer<typeof productSchema>
export type ProductStatus = z.infer<typeof productStatusSchema>
export type ProductIdentity = z.infer<typeof productIdentitySchema>
```

**Scenario 3 — input schema: the model schema says what a product *is*, this says what a human is allowed to type:** `[SCH-9, SCH-10, SCH-L5, SCH-4, CTL-L2]`

```typescript
import {
  DateOnlySchema,
  MoneySchema,
  OptionalStringSchema,
  RequiredArraySchema,
  RequiredStringSchema,
  UrlSchema,
} from '@turystack/fields'
import { CnpjSchema } from '@turystack/fields/br'
import { createRequestSchema } from '@turystack/nestjs-server'
import { z } from 'zod'

import { productSchema, productStatusSchema } from '@/domains/product/product.schema'

export const createProductRequest = createRequestSchema({
  body: z.object({
    // SCH-L5: '   ' fails as `required`, not as `too_small`; the value is
    // sanitized before the bounds are measured
    name: RequiredStringSchema({ min: 3, max: 120 }),
    description: OptionalStringSchema({ max: 2000 }),
    // SCH-L5: minor units assembled from the digit string — no amount is ever
    // rounded through a float on its way in
    priceCents: MoneySchema({ minCents: 1 }),
    // SCH-L5: a calendar date stays a calendar date; z.coerce.date() would make
    // it an instant and move the day in a negative-offset zone
    availableOn: DateOnlySchema({ notPast: true }),
    supplierDocument: CnpjSchema(),
    manualUrl: UrlSchema({ blockPrivateHosts: true, requireHttps: true }),
    categoryIds: RequiredArraySchema(productSchema.shape.categoryId),
    status: productStatusSchema,
  }),
})
```

**Scenario 4 — the failure crosses the wire as a code:** `[SCH-10]`

```typescript
import { formatErrors } from '@turystack/fields'
import type { FieldIssueCode } from '@turystack/fields'

// SCH-10: exhaustive by construction — a new code that has no message is a
// compile error here, not a blank string in production
const messages: Record<FieldIssueCode, string> = { /* ... */ }

const result = createProductRequest.body.safeParse(input)

if (!result.success) {
  const errors = formatErrors(result.error, (issue) => messages[issue.code])
  // { name: { code: 'required', message: 'Campo obrigatório', params: {}, path: 'name' } }
}
```

### ❌ Never do

```typescript
// ❌ [SCH-L1] importing from zod/v4
import { z } from 'zod/v4'

// ❌ [SCH-4] field without .meta / verbose description
name: z.string()
createdAt: z.date().meta({ description: 'Timestamp when the product was created' })

// ❌ [SCH-L2] date as a string
createdAt: z.string()

// ❌ [SCH-5] inline enum instead of an extracted named schema
status: z.enum(['ACTIVE', 'INACTIVE'])

// ❌ [ARC-CTR-2] duplicating the related schema instead of importing it
category: z.object({ categoryId: z.string(), name: z.string() })

// ❌ [ARC-CTR-3] FK join with the full entity instead of the Identity (heavy/recursive join)
category: categorySchema  // use categoryIdentitySchema

// ❌ [SCH-1] hand-written type instead of derived from the schema
export type Product = { productId: string; name: string }

// ❌ [SCH-2] schema and type in the same file

// ❌ [SCH-L3] custom wrapper instead of native Zod
const myString = (desc: string) => z.string().meta({ description: desc })
field: myString('Product name')  // use z.string().meta({ description: 'Product name' })

// ❌ [SCH-L5] hand-rolled input validation — every line here has a case it misses
name: z.string().min(1).trim()          // '   ' passes — .trim() runs after the bound
name: z.string().trim().min(1)          // right order, wrong code: reports `too_small`,
                                        // and a zero-width space passes either way
                                        // use RequiredStringSchema()
priceCents: z.coerce.number()           // '' becomes 0 — a blank field is not a zero
                                        // use MoneySchema()
availableOn: z.coerce.date()            // '2026-01-01' is 31 December in São Paulo
                                        // use DateOnlySchema()
document: z.string().regex(/^\d{11}$/)  // 111.111.111-11 matches and is invalid
                                        // use CpfSchema()
manualUrl: z.url()                      // accepts javascript: and http://169.254.169.254
                                        // use UrlSchema({ blockPrivateHosts: true })

// ❌ [SCH-10] branching on the message text instead of the code
if (issue.message.includes('Too small')) { /* breaks on the next zod release */ }

// ❌ [SCH-L4] metadata in a comment or a separate object instead of .meta() chained on the field
/** @description Product name */
name: z.string()
```
