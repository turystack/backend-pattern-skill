# Project Structure & Domain Boundaries

**Concept.** The structure reflects business capabilities and delivery points.
Folders exist when they hold real code; the CLI does not create empty zones.

## Invariants

| ID | Law |
|---|---|
| PRJ-2 | Dependency order: schema → entity → repository → use-case → controller/handler. |
| PRJ-6 | Global `@turystack/*` modules are registered once at the consuming app's root. |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-3` | Organization by domain/feature. |
| `ARC-LAY-2` | Domain does not import infra. |
| `ARC-LAY-4` | Cross-domain through the public operation. |
| `ARC-LAY-6` | No mandatory technical-layer folder. |
| `ARC-4` | The helper lives with its owner. |
| `ARC-6` | The delivery boundary is thin. |
| `ARC-TOP-3` | The app registers only the closure it consumes. |
| `ARC-15` | Config validated at boot. |


## Standalone API

```text
src/
├── controllers/
│   └── order/
│       ├── order.controller.ts
│       └── order.schemas.ts
├── domains/
│   └── order/
│       ├── order.schema.ts
│       ├── order.types.ts
│       ├── order.entity.ts
│       ├── order.repository.ts
│       ├── order.mock.ts
│       ├── use-cases/
│       │   └── create-order/
│       │       ├── create-order.ts
│       │       ├── create-order.types.ts
│       │       └── create-order.test.ts
│       └── index.ts
├── database/                 # only with @turystack/nestjs-database
│   ├── database.schema.ts
│   └── database.relations.ts
├── support/                  # optional: pure cross-cutting utilities
├── app.module.ts
├── config.schema.ts
├── exceptions.ts
└── main.ts
```

- `adapters/` is born only when an integration not covered by the libs exists.
- `database/`, migrations, scripts and local services are born only when a
  database is selected.
- `app.module.ts` registers controllers, the provider closure and global modules.
- `main.ts` contains only `Server.create`.
- A multi-project API adds only the
  `controllers/{project}/{domain}/` level.

## Monorepo

```text
apps/
├── api/
│   └── src/
│       ├── controllers/
│       ├── app.module.ts
│       ├── config.schema.ts
│       └── main.ts
└── process-payment/
    └── src/
        ├── process-payment.handler.ts
        ├── process-payment.module.ts
        ├── process-payment.schemas.ts
        ├── config.schema.ts
        └── main.ts
libs/
├── domains/
│   └── src/
│       ├── order/
│       ├── exceptions.ts
│       └── index.ts
├── database/                 # only when a database is selected
│   ├── src/
│   │   ├── database.schema.ts
│   │   ├── database.relations.ts
│   │   └── index.ts
│   ├── drizzle/
│   └── drizzle.config.ts
└── support/                  # optional package, only if genuinely cross-app
```

- `@repo/domains` holds the product's entities, repositories, use-cases, events
  and exceptions.
- `@repo/database` holds the schema, relations, migrations and the
  `DatabaseService` augmentation.
- APIs and handlers are delivery apps: they compose modules and import
  operations from `@repo/domains`.
- Each app has its own configuration and registers only the libs it needs.

## Domain anatomy

Each file has a role. An exclusive 1:1 sub-resource belongs to the aggregate; it
does not become a separate domain. A use-case lives in `use-cases/{operation}/`.
The barrel exposes only the domain's public API.

A local repository is created when it adds domain policy or composition: a
multi-table aggregate, a specific query, soft delete or entity hydration. For
trivial operations already covered by the `DatabaseService` typed repository, do
not add an interface and a wrapper with no behavior.

`support/` is allowed for pure functions with no domain owner — for example,
cross-cutting normalization used by several domains and still specific to the
product. It takes no business rule, infrastructure client, logger,
configuration or lib wrapper. In a monorepo, code shared across apps is born as
an explicit package (`libs/support`) only once the reuse exists.

## Dependency injection

- Global lib → service/token provided by the lib itself.
- Swappable local adapter → interface + `Symbol`.
- Use-case → class.
- Controller/handler → use-case.
- Use-case → its own repository, other domains' use-cases and lib/adapter
  services.

Use package/alias imports; never a relative path crossing a domain. Follow the
ordering configured by `@turystack/backend-config`.

## Never do

- Create empty folders to represent layers.
- Create `src/infrastructure/cache` or an adapter around a Turystack lib.
- Use `src/support` as a generic destination for ownerless code.
- Create `OrderModule` just to re-export the domain's providers.
- Put a shared schema/migration inside a monorepo app.
- Read `process.env` in a controller, use-case, repository, adapter or `main.ts`.
