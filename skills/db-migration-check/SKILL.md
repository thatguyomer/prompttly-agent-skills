---
name: db-migration-check
description: Audits a database migration for operations that lock tables, break running code, or fail to roll back. Checks for blocking index builds, non-nullable columns added without defaults, renames and drops that break the currently deployed release, missing indexes on new foreign keys, and unbounded backfills. Produces a safe rewrite and a deployment order. Use when reviewing or writing a schema migration, or before running one against production. Do not use for query performance tuning of existing tables or for ORM model design questions.
---

# Database Migration Safety Check

Most migrations are fine in development and catastrophic in production, because
development has a hundred rows and production has fifty million. This skill
checks a migration for the difference.

## When to use this skill

- Reviewing a migration in a pull request
- Writing a migration against a table with meaningful row counts
- Before running a migration against production

## When not to use it

- Tuning slow queries on an unchanged schema
- ORM modelling or relationship design questions
- Seed or fixture scripts that run against empty databases

## Inputs

- The migration file, up and down
- The target engine and version — Postgres 11+ and 9.x differ materially here
- Approximate row count of each affected table, if known. Ask; do not guess.

## The core rule

**A migration must be safe to run while the previous release is still serving
traffic.** Deploys are not atomic. For some window, old code talks to the new
schema. Every check below follows from that.

## Procedure

### 1. Locking

| Operation | Risk | Safe alternative |
|---|---|---|
| `CREATE INDEX` | Locks writes for the build | `CREATE INDEX CONCURRENTLY` (outside a transaction) |
| `ADD COLUMN` with volatile default | Rewrites the table | Add nullable, backfill in batches, then set default |
| `ALTER COLUMN TYPE` | Rewrites and locks | New column, dual-write, backfill, swap |
| `ADD CONSTRAINT ... CHECK` | Full scan under lock | `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT` |
| `ADD FOREIGN KEY` | Locks both tables | Add `NOT VALID`, validate separately |

Postgres 11+ makes `ADD COLUMN ... DEFAULT <constant>` cheap. A *volatile*
default such as `now()` or `gen_random_uuid()` still rewrites the table.

### 2. Compatibility with the running release

- **Dropping a column** the deployed code still selects: breaks immediately.
  Ship a release that stops referencing it, then drop it in a later migration.
- **Renaming** anything: this is a drop plus an add. Never rename in one step.
  Add the new name, dual-write, migrate readers, then drop.
- **Adding `NOT NULL` without a default**: every insert from the old code fails,
  because the old code does not know to supply the column.

### 3. Indexes on new relationships

Every new foreign key column needs an index. Postgres indexes the referenced
primary key, not the referencing column, so `ON DELETE CASCADE` and every join
through it will sequentially scan without one.

### 4. Backfills

- Unbounded `UPDATE` over a large table holds locks and inflates WAL.
- Batch it: bounded loop, explicit `LIMIT`, commit between batches.
- A backfill belongs in its own migration, separate from the schema change.

### 5. Reversibility

- Is there a `down`, and does it actually restore the prior state?
- A `down` that drops a column populated by the `up` destroys data. Say so
  explicitly rather than treating the migration as reversible.

## Output contract

```
### Verdict
SAFE | SAFE_WITH_CHANGES | UNSAFE

### Findings
- [severity] operation — what happens in production at the stated row count
  Fix: the concrete rewrite

### Rewritten migration
<the corrected SQL, or "no changes needed">

### Deployment order
1. ...
2. ...
```

State the assumed row count in the verdict. If it was never provided, say
`row count unknown — assessed as if large` rather than assuming small.

## Worked example

**Input:**

```sql
ALTER TABLE orders ADD COLUMN customer_id uuid NOT NULL;
CREATE INDEX idx_orders_customer ON orders (customer_id);
```

**Expected output:**

```
### Verdict
UNSAFE (orders assumed large; row count unknown)

### Findings
- [blocking] ADD COLUMN ... NOT NULL with no default — fails outright if the
  table has rows, and every insert from the currently deployed release fails
  because it does not supply customer_id.
  Fix: add the column nullable, backfill, add the constraint separately.
- [blocking] CREATE INDEX without CONCURRENTLY — holds a write lock on orders
  for the duration of the build.
  Fix: CREATE INDEX CONCURRENTLY, outside a transaction.

### Rewritten migration
-- migration 1
ALTER TABLE orders ADD COLUMN customer_id uuid;

-- migration 2 (no transaction)
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders (customer_id);

-- migration 3, after the backfill completes
ALTER TABLE orders
  ADD CONSTRAINT orders_customer_id_not_null
  CHECK (customer_id IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_id_not_null;

### Deployment order
1. Migration 1 and 2.
2. Deploy code that writes customer_id on every insert.
3. Backfill existing rows in batches.
4. Migration 3.
```

## Common mistakes

- Treating `CREATE INDEX CONCURRENTLY` as a drop-in replacement: it cannot run
  inside a transaction, and most migration tools wrap statements in one by
  default. Disable it for that migration.
- Assuming a migration that is fast locally is fast in production.
- Writing a `down` that silently loses data and calling it reversible.
