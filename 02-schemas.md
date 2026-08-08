# Schemas

**Concept.** The schema is the **source of truth** for the domain's types. It mirrors the data model, and **every** type derives from it: the entity's fields, the repository's inputs and the controller's DTOs. It is the base of the stack — `Schema → Entity → Repository → UseCase → Controller`.

> **How to read this file.** Two parts, deliberately separated:
> - **🌐 Generic pattern** — the **portable law**. Holds in any stack (NestJS, PHP, etc.); the text stays true even after swapping frameworks. This is what review enforces as an invariant.
> - **🛠️ Project-specific** — the **code** that implements each rule in the current stack (TypeScript · NestJS · @turystack · Zod). Swapping stacks rewrites **only** this part; the law above does not change.
>
> Each rule carries an id `SCH-n` linking the law (generic) to the code (specific). Gates bind by **id**, not by file/line. `SCH-L*` rules are **stack lints**: they exist only because of the current language's ergonomics (they invert in another stack), but they stay id-registered and enforced in this project.

---

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

### Invariants (the law the gates enforce)

> Bind by **id**. `constitutional` = portable invariant (holds in any stack). `stack lint` = exists only because of the current language's ergonomics and inverts in another stack — still id-registered and enforced here. The **detector** (how the gate catches the violation) is project-specific and lives in the 🛠️ part below.

| ID | Law (one line) | Type | Detector (🛠️) |
|---|---|---|---|
| SCH-1 | Schema is the source of truth; downstream types derived by inference/projection, never hand-written | constitutional | Scenarios 1–2 / ❌ |
| SCH-2 | Schemas and derived types in separate files, each with an exclusive responsibility; never mixed | constitutional | Scenarios 1–2 / ❌ |
| SCH-3 | Schema covers every field of the model (PK, FKs, business, timestamps, audit) | constitutional | Scenario 1 / ❌ |
| SCH-4 | Every field carries inline metadata (description + example); terse descriptions | constitutional | Scenarios 1–2 / ❌ |
| SCH-5 | Enum/complex type extracted as a named schema before the main schema; never inline | constitutional | Scenarios 1–2 / ❌ |
| SCH-8 | Field ordering: PK → FKs (+ Identity) → important → booleans → status → timestamps + audit | constitutional | Scenario 1 |
| SCH-L1 | Import from `zod`, never from `zod/v4` | stack lint | ❌ |
| SCH-L2 | Dates use `z.date()` (never `z.string()`); optional/nullable uses `.nullish()` | stack lint | Scenario 1 / ❌ |
| SCH-L3 | 100% native Zod, no wrappers; `z.infer` to derive types (never hand-write the shape) | stack lint | Scenarios 1–2 / ❌ |
| SCH-L4 | Metadata via `.meta({ description, example })` chained directly on the field | stack lint | Scenarios 1–2 / ❌ |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-CTR-1` | Contract declared once, types derive. |
| `ARC-CTR-2` | Relationship imported from the owning module. |
| `ARC-CTR-3` | Module exposes an Identity to be referenced. |
| `ARC-CTR-7` | Contract data stays derived; never copied into local state. |

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
- **SCH-L3** — use `z.object`, `z.string`, `z.uuid`, `z.boolean`, `z.array`, `z.enum` directly; no custom wrappers over Zod.
- **SCH-L4** — `.meta(...)` chained directly on the field's schema (never in a comment nor in separate docs).

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

// ❌ [SCH-L4] metadata in a comment or a separate object instead of .meta() chained on the field
/** @description Product name */
name: z.string()
```
