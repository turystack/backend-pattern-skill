# Repositories & Persistence Policy

**Concept.** `DatabaseService` already provides typed repositories per table. A
domain repository only exists when it adds policy or composition that the typed
API cannot express on its own; it is not a mandatory ceremonial layer.

## Decision

Use the `DatabaseService` typed repository directly in the use-case when the
operation is simple single-table CRUD, with no hydration, soft delete or
reusable query of the domain's own.

Create `{domain}.repository.ts` when at least one of these reasons applies:

- an aggregate spanning multiple tables with its own transactional boundary;
- a mandatory filter, such as soft delete or tenant scope;
- row → entity/identity hydration;
- a composed query reused by more than one operation;
- a concurrency, lock or consistency policy;
- a real need to replace the persistence behind a local contract.

## Invariants

| ID | Law |
|---|---|
| REP-1 | The repository uses data verbs (`find`, `save`, `update`), never business operation names. |
| REP-2 | The business rule stays in the use-case/entity; the repository only applies persistence policy. |
| REP-3 | If the domain adopts soft delete, every read applies the filter and removal uses an update; there is no accidental physical delete. |
| REP-4 | A multi-table aggregate is persisted by a root repository and a transaction local to the aggregate. |
| REP-5 | Hydration is explicit and consistent: the main composition complete, the related FK as an Identity when needed. |
| REP-7 | Inputs derive from the canonical schemas; filters/pagination use `@turystack/query-dsl`. |
| REP-8 | A recurring query needs a coherent index and an integration test; an index is not added on intuition. |
| REP-9 | A schema change is delivered by a versioned migration compatible with the rollout. |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-CON-1` | Transactional boundary declared. |
| `ARC-CON-2` | Transaction sized to the atomicity required. |
| `ARC-CTR-6` | Schema change compatible with the rollout. |


## Local contract

When a domain repository is justified:

- export an interface + `Symbol` only if replacement/test doubles via DI are
  necessary; otherwise inject the class directly;
- keep input types file-local when they are not part of the domain's API;
- make `findById` reflect the chosen contract consistently (`Entity` +
  not-found or `Entity | null`);
- map fields explicitly; do not use a cast or an automatic parser to hide the
  difference between row and entity;
- keep query helpers private at the end of the class.

```typescript
@Injectable()
export class OrderRepository {
  constructor(private readonly db: DatabaseService) {}

  async findById(orderId: string) {
    const row = await this.db.orders.findFirst({
      where: (fields, { and, eq, isNull }) =>
        and(eq(fields.orderId, orderId), isNull(fields.deletedAt)),
      with: ORDER_WITH,
    })

    if (!row) {
      throw new exceptions.order.notFound({ orderId })
    }

    return this.parseOrder(row)
  }

  private parseOrder(row: OrderRow) {
    return new OrderEntity({
      orderId: row.orderId,
      status: row.status,
      items: row.items.map((item) => new OrderItemEntity(item)),
    })
  }
}
```

## Database placement

- Standalone: `src/database/`, only when a database has been selected.
- Monorepo: `libs/database/`, shared by the apps.
- Schema, relations, migrations, config and augmentation follow the
  `@turystack/nestjs-database` documentation; this skill does not replicate its API.

Before adding a migration, assess expand/contract compatibility, backfill, lock
and operational rollback. Do not mix a destructive change with code that still
depends on the old column.

## Never do

- Create an interface + wrapper for every table by default.
- Duplicate methods already provided by `DatabaseService`.
- Create `activateOrder`, `cancelOrder` or any other business verb in the repository.
- Access another domain's repository; call that domain's public use-case.
- Run a raw query to work around typing without justification and a test.
- Create `infrastructure/database` around the library.
