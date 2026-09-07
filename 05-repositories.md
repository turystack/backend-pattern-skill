# Repositories & Persistence Policy

**Concept.** `DatabaseService` already provides typed repositories per table. A
domain repository only exists when it adds policy or composition that the typed
API cannot express on its own; it is not a mandatory ceremonial layer.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> @turystack · Drizzle. The split, `XXX-n` versus `XXX-Ln`, and why an `ARC-…`
> law is cited and never restated: `turystack-backend-pattern` › *How a
> section is written*.

---

**Rules defined here:** `REP-1` · `REP-2` · `REP-3` · `REP-4` · `REP-5` ·
`REP-8` · `REP-9` · `REP-L1` — the law is the *Invariants* table below; every
❌ item cites the id it violates.

**Retired ids:** `REP-6` · `REP-7` — retired, not renumbered. A review or
commit citing one points at a rule that no longer exists; the number is never
reused.

## 🌐 Generic pattern (portable — stack-independent)

### Decision — does this repository exist at all?

Use the typed repository the database layer already provides when the operation
is simple single-table CRUD, with no hydration, soft delete or reusable query of
the domain's own.

Create `{domain}.repository.ts` when at least one of these reasons applies:

- an aggregate spanning multiple tables with its own transactional boundary;
- a mandatory filter, such as soft delete or tenant scope;
- row → entity/identity hydration;
- a composed query reused by more than one operation;
- a concurrency, lock or consistency policy;
- a real need to replace the persistence behind a local contract.

None of the six applies → the repository is ceremony. An interface plus a
wrapper that only forwards a call adds a file to read, a mock to maintain and no
decision (`ARC-LAY-8`).

### Invariants (the law the gates enforce)

| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| REP-1 | The repository uses data verbs (`find`, `save`, `update`), never business operation names | constitutional | `gate:no-business-verb-in-repo` | Local contract / ❌ |
| REP-2 | The business rule stays in the use-case/entity; the repository applies persistence policy only | constitutional | `manual` | Local contract / ❌ |
| REP-3 | With soft delete, every read applies the filter and removal is an update; no accidental physical delete | constitutional | `gate:soft-delete-filter` | Local contract / ❌ |
| REP-4 | A multi-table aggregate is persisted by a root repository, in a transaction local to the aggregate | constitutional | `manual` | Local contract |
| REP-5 | Hydration is explicit and consistent: the main composition complete, a related FK as an Identity | constitutional | `manual` | Local contract / ❌ |
| REP-8 | A recurring query needs a coherent index and an integration test; an index is never added on intuition | constitutional | `manual` | Database placement |
| REP-9 | A schema change ships as a versioned migration compatible with the rollout | constitutional | `gate:migration-shape` | Database placement / ❌ |
| REP-L1 | Inputs derive from the canonical schemas; filters, sorting and pagination go through `@turystack/query-dsl` | stack lint | `grit:no-hand-rolled-filter` | Local contract / ❌ |

Ids are stable across versions; a gap is a law that moved to the constitution.
`REP-7` became `REP-L1` — it names a library, so it is a stack lint, not a
portable law.

**Why REP-1 forbids `cancelOrder` in a repository.** A business verb in the
persistence layer puts the same rule in two places: the entity that owns the
transition and the query that performs it. The next change touches one and
forgets the other, and the two disagree with no test failing.

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-CON-1` | Every write declares its consistency strategy. | `@Transactional()` on the operation that owns the boundary |
| `ARC-CON-2` | A transaction is the size of the atomicity required. | the aggregate's root repository, not the whole use-case |
| `ARC-CON-4` | A transaction is not a concurrency tool. | constraint, version column or `@turystack/nestjs-lock` |
| `ARC-CTR-6` | A contract-breaking change ships compatible with the rollout. | expand → backfill → contract, in separate migrations |
| `ARC-LAY-8` | Infra covered by a lib is used directly, never wrapped. | `DatabaseService` injected; no `infrastructure/database` |

---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Drizzle)

### Local contract

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

### Database placement

- `packages/database`, shared by every app and by the migration CLI. It is a
  package rather than a folder in the API because a handler app, a second API
  and drizzle-kit all read the same schema.
- Schema, relations, migrations, config and augmentation follow the
  `@turystack/nestjs-database` documentation; this skill does not replicate its API.

Before adding a migration, assess expand/contract compatibility, backfill, lock
and operational rollback. Do not mix a destructive change with code that still
depends on the old column.

### Worked example · renaming a column without a window

A rename is the smallest change that can take an API down, because for a few
minutes two versions of the code are live at once and they disagree about the
schema. `ARC-CTR-6` says both versions coexist while a consumer remains, and
`REP-9` says the migration ships compatible with the rollout. That means one
rename is **three deploys**, never one:

```text
                    database                    code that is live
expand    ①  add `full_name`, nullable      old code: writes `name`
             no backfill yet                new code: writes both, reads `name`
             ↑ safe to roll back — nothing reads the new column yet

backfill  ②  copy name → full_name          same code as ①
             in batches, no table lock       reads still served by `name`
             ↑ safe to re-run; it is idempotent by construction

contract  ③  drop `name`                    code: writes and reads `full_name`
             after the backfill is verified  and no longer mentions `name`
             ↑ the only irreversible step, and it ships alone
```

Three properties make it survivable, and skipping any one of them is what turns
a rename into an incident:

- **Every intermediate state is deployable.** At no point does live code depend
  on a column that does not exist yet, or on one that was just dropped.
- **The backfill is restartable.** It processes in bounded batches
  (`ARC-DEL-6`), and running it twice produces the same result — which is what
  lets you stop it mid-flight when the load says so.
- **The destructive step is alone in its deploy.** Bundled with a feature, its
  rollback drags the feature back with it, so nobody rolls back and the incident
  gets longer.

Adding a `NOT NULL` column follows the same shape: add nullable, backfill,
*then* add the constraint. Adding it non-null in one step locks the table and
fails the moment an old row cannot satisfy it.

### ❌ Never do

- `[ARC-LAY-8]` Create an interface + wrapper for every table by default.
- `[ARC-LAY-8]` Duplicate methods already provided by `DatabaseService`, or create `infrastructure/database` around the library.
- `[REP-1]` Create `activateOrder`, `cancelOrder` or any other business verb in the repository.
- `[REP-2]` Decide a business rule inside a query instead of in the entity.
- `[ARC-LAY-4]` Access another domain's repository; call that domain's public use-case.
- `[REP-3]` Read a soft-deleted domain without the filter, or delete its row physically.
- `[REP-5]` Hide the row → entity difference behind a cast or an automatic parser.
- `[REP-L1]` Hand-roll filter/sort/pagination parsing instead of `@turystack/query-dsl`.
- `[REP-9]` Ship a destructive migration together with code that still reads the old column.
- `[REP-8]` Run a raw query to work around typing without justification and a test.
