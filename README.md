# Expand-Contract Pattern

## The Two Coexisting Schemas

There is only **one physical schema** — the real tables — but the database presents **two logical schemas** at the same time:

- **The old logical schema** — what the previous application version expects: the original column names, types, and constraints
- **The new logical schema** — what the next application version expects: the renamed column, the new type, the added constraint, etc.

Old pods talk to the old logical schema, new pods talk to the new one, and the database (or application) keeps the data they see consistent

Typically implemented with **versioned views** backed by the same underlying tables, each exposing the data shape the corresponding application version expects

A pod selects its logical schema once at connection time (e.g. via `search_path`) and is unaffected by the rollout from then on

## The Three Phases

### 1. Expand

Add the new structure **alongside** the old one, never in place of it

Every operation in Expand is chosen so it can run on a busy table without holding long locks

During the rollout window, a write can arrive through either logical schema; dual-writes guarantee that both representations stay consistent

Dual-writes cover _new_ writes; the backfill covers everything that already existed

- Add new columns or tables; leave existing columns and tables untouched
- Build the new logical schema (the new view) so next-version pods can query it immediately, even if no row has been migrated yet
- Dual-write to both columns via a trigger (or in application code) to keep the old and new columns in sync
- Backfill existing rows in batches so the new column is populated for historical data

After Expand finishes, both logical schemas are live and writable

Any mix of old-version and new-version application instances can run against the database

### 2. Roll Out the Application

The application rollout is now a **completely independent operation**: the new binary can be deployed gradually

The database is not involved — both code paths still work because both logical schemas still work

### 3. Contract

Once every running instance has moved to the new version, the old structure is removed:

- Drop the dual-write trigger (or the equivalent logic in application code)
- Drop the obsolete columns or tables
- Drop the old logical schema (the old view)

After Contract, only the new shape remains

The transition is complete

# Migrations Guideline

## Pre-Deploy vs Post-Deploy

Migrations split into two phases around the application rollout:

- **Pre-deploy** runs before the new application version is deployed; use it for additive, backward-compatible changes
- **Post-deploy** runs after every instance has moved to the new version; use it for destructive or restrictive changes

## Locking

DDL that takes `ACCESS EXCLUSIVE` is "fast" in isolation but can stall the database if it queues behind a long-running query: while the migration waits, every subsequent reader queues _behind_ it

Two defenses, applied to every DDL session:

1. Query [`pg_locks`](https://www.postgresql.org/docs/18/view-pg-locks.html) for long-running blockers on the target table; pause and retry until the queue is short
2. Bound the wait with `lock_timeout`:

   ```sql
   BEGIN;
   SET LOCAL lock_timeout = '2s';   -- fail if not acquired in 2s
   ALTER TABLE t ADD COLUMN c text;
   COMMIT;
   ```

Operations that take a _weaker_ lock — `CREATE INDEX CONCURRENTLY`, `ALTER TABLE … VALIDATE CONSTRAINT` — don't block concurrent reads or writes, so they don't need this protection

They can still wait a long time for in-flight transactions on the table to drain, but that wait is invisible to other clients

## Migrations and Transactions

Some migrations must run outside a transaction:

- `CREATE INDEX CONCURRENTLY`, `DROP INDEX CONCURRENTLY`, `REINDEX CONCURRENTLY`
- Batched data migrations (backfills)

## Table Operations

### Create a Table

[`CREATE TABLE`](https://www.postgresql.org/docs/18/sql-createtable.html) does not lock existing tables

For FKs, follow [Add a `FOREIGN KEY`](#add-a-foreign-key)

### Rename a Table

[`ALTER TABLE … RENAME`](https://www.postgresql.org/docs/18/sql-altertable.html) takes a short `ACCESS EXCLUSIVE`

1. Rename the table and create a view under the old name:

   Reads, inserts, updates, and deletes against the view continue to work — see the [updatable views caveats](https://www.postgresql.org/docs/18/sql-createview.html#SQL-CREATEVIEW-UPDATABLE-VIEWS)

   ```sql
   BEGIN;
   ALTER TABLE t_old RENAME TO t_new;
   CREATE VIEW t_old AS SELECT * FROM t_new;
   COMMIT;
   ```
2. Deploy a code change to start using the new table
3. Drop the temporary view:

   ```sql
   DROP VIEW t_old;
   ```

Sanity-check via [`pg_stat_user_tables`](https://www.postgresql.org/docs/18/monitoring-stats.html#MONITORING-PG-STAT-USER-TABLES-VIEW) that the view is no longer being read before dropping it

### Drop a Table

`DROP TABLE` takes `ACCESS EXCLUSIVE`

If another table has a FK pointing to this one, drop those FKs first in separate migrations (one FK per migration); otherwise `DROP TABLE` fails

Avoid `DROP TABLE … CASCADE` — it acquires `ACCESS EXCLUSIVE` on every dependent table in a single transaction, increasing the chance of deadlocks

1. Deploy a code change to stop using the table
2. Drop the now unused table:

   ```sql
   DROP TABLE t;
   ```

Sanity-check via `pg_stat_user_tables` that the table is no longer being read before running the post-deploy migration

## Columns

### Add a Column

`ALTER TABLE … ADD COLUMN` takes `ACCESS EXCLUSIVE`; the duration depends on whether the table needs a rewrite

#### Without a Rewrite

No rewrite is required when the new column is one of:

- No `DEFAULT` specified (the column defaults to `NULL`)
- A non-volatile `DEFAULT` (e.g., a constant or a non-volatile expression)
- A virtual generated column

For a non-volatile `DEFAULT`, the value is evaluated once at statement time and stored in the table's metadata; existing rows return the default on access without being rewritten

```sql
ALTER TABLE t ADD COLUMN c text;                                      -- defaults to NULL
ALTER TABLE t ADD COLUMN c text DEFAULT 'foo';                        -- non-volatile DEFAULT
ALTER TABLE t ADD COLUMN c int GENERATED ALWAYS AS (a + b) VIRTUAL;   -- virtual generated
```

#### With a Rewrite

The entire table and its indexes are rewritten under `ACCESS EXCLUSIVE` when the new column has any of:

- A volatile `DEFAULT` (e.g., `clock_timestamp()`, `random()`, `nextval()`)
- `GENERATED ALWAYS AS (…) STORED`
- `GENERATED … AS IDENTITY`
- A domain data type that has constraints

For a volatile `DEFAULT`, skip the rewrite by adding the column nullable first, backfilling, and then attaching the default for future inserts:

1. Add the column as nullable with no `DEFAULT`
2. Backfill the desired values in batches, outside any transaction
3. Set the original `DEFAULT` for future inserts:

   Applies only to subsequent `INSERT` and `UPDATE` statements and does not change rows already in the table, regardless of the expression's volatility

   ```sql
   ALTER TABLE t ALTER COLUMN c SET DEFAULT …;
   ```
4. If the column needs to be `NOT NULL`, follow [Add `NOT NULL` constraint](#add-a-check-or-not-null-constraint)

### Drop a Column

`DROP COLUMN` takes a short `ACCESS EXCLUSIVE` while the column is marked dropped in the catalog

1. Deploy a code change to stop using the column
2. Drop the now unused column:
   1. Find dependents in the [`pg_depend`](https://www.postgresql.org/docs/18/catalog-pg-depend.html) catalog
   2. Drop dependent indexes with `DROP INDEX CONCURRENTLY` first; otherwise they would be dropped along with the column under `ACCESS EXCLUSIVE`
   3. If a view references the column, recreate the view without it
   4. Drop the column:

      ```sql
      ALTER TABLE t DROP COLUMN c;
      ```

Sanity-check via `pg_stat_user_tables` that the column is no longer being read before running the post-deploy migration

### Rename a Column

Both names must coexist while the deploy is in progress, because old code still uses the old name and new code uses the new one

1. Create a new temporary column with a target name:
   1. Add the new column
   2. Dual-write to both columns via a `BEFORE INSERT/UPDATE` trigger (or in application code)
   3. [Build indexes](#create-an-index) that referenced the old column on the new column
   4. [Recreate any FKs](#add-a-foreign-key) that referenced the old column on the new column
   5. If a view references the old column, recreate it pointing at the new column
   6. Backfill existing data from the old column to the new column in batches, outside any transaction
2. Deploy a code change to stop using the old column
3. Drop the trigger and the now unused old column

### Change a Column's Type

#### Without a Rewrite

If the change is catalog-only, it is a single migration with a short `ACCESS EXCLUSIVE`, without a table rewrite

Verify that [`pg_class.relfilenode`](https://www.postgresql.org/docs/18/catalog-pg-class.html) doesn't change before treating it as catalog-only

```sql
ALTER TABLE t ALTER COLUMN c TYPE text;
```

Indexes on the column may still be rebuilt under the same `ACCESS EXCLUSIVE` unless PostgreSQL can prove the new index is logically equivalent to the old one (e.g. collation unchanged); check that each index's `relfilenode` is unchanged to confirm

#### With a Rewrite

For changes that rewrite the column, the approach is almost identical to [renaming a column](#rename-a-column)

Validate the cast on a clone first — a failed cast mid-backfill leaves the column half-migrated

1. Create a new temporary column with a target type:
   1. Add a new column of the target type
   2. Dual-write to both columns via a `BEFORE INSERT/UPDATE` trigger (or in application code), casting in both directions
   3. [Build indexes](#create-an-index) that referenced the old column on the new column
   4. [Recreate any FKs](#add-a-foreign-key) that referenced the old column on the new column
   5. If a view references the old column, recreate it pointing at the new column
   6. Backfill existing data with the cast, in batches, outside any transaction
2. Deploy a code change to stop using the old column
3. Drop the trigger and the now unused old column

## Indexes

### Create an Index

[`CREATE INDEX CONCURRENTLY`](https://www.postgresql.org/docs/18/sql-createindex.html) takes `SHARE UPDATE EXCLUSIVE`

It cannot run inside a transaction (see [Migrations and Transactions](#migrations-and-transactions)) and requires two table scans

It also waits for in-flight transactions on the table to complete before finishing — it can stall behind a long query, but that wait does not block any other client

```sql
CREATE INDEX CONCURRENTLY t_idx ON t (c);
```

If the build fails, the database leaves an `INVALID` index — drop it before retrying:

```sql
DROP INDEX CONCURRENTLY IF EXISTS t_idx;
```

### Drop an Index

`DROP INDEX CONCURRENTLY` takes `SHARE UPDATE EXCLUSIVE`

```sql
DROP INDEX CONCURRENTLY t_idx;
```

Cannot drop an index backing a `PRIMARY KEY` or `UNIQUE` constraint — drop the constraint instead, which removes the index

### Reindex an Index

Same `SHARE UPDATE EXCLUSIVE` lock and caveats as for [`CREATE INDEX CONCURRENTLY`](#create-an-index)

```sql
REINDEX INDEX CONCURRENTLY t_idx;
```

## Constraints

### Add a `CHECK` or `NOT NULL` Constraint

[`ADD … NOT VALID`](https://www.postgresql.org/docs/18/sql-altertable.html) registers the constraint under a short `ACCESS EXCLUSIVE` without scanning existing rows

[`VALIDATE CONSTRAINT`](https://www.postgresql.org/docs/18/sql-altertable.html) scans them later under `SHARE UPDATE EXCLUSIVE`

New rows are checked from the moment the constraint is added

`ADD … NOT VALID` and `VALIDATE CONSTRAINT` must run in separate transactions — otherwise the validation scan inherits the `ACCESS EXCLUSIVE` held since `NOT VALID`, blocking the table for the full scan

PostgreSQL 18 supports the same `ADD … NOT VALID` / `VALIDATE CONSTRAINT` pattern for `NOT NULL`, replacing the pre-18 `SET NOT NULL` form that required a table scan

1. Deploy a code change to stop writing rows that would violate the constraint
2. Register the not-valid constraint:

   ```sql
   -- CHECK
   ALTER TABLE t ADD CONSTRAINT t_c CHECK (c > 0) NOT VALID;
   
   -- NOT NULL (PG 18+)
   ALTER TABLE t ADD CONSTRAINT t_c NOT NULL c NOT VALID;
   ```
3. Fix existing violating rows in batches, outside any transaction
4. Validate the constraint on existing rows:

   ```sql
   ALTER TABLE t VALIDATE CONSTRAINT t_c;
   ```

After `VALIDATE` for `NOT NULL`, the column behaves as if declared `NOT NULL` at column level — the optimizer treats it the same, and no follow-up `SET NOT NULL` step is needed

For partitioned tables, validate each partition first, then attach to the parent

### Add a `FOREIGN KEY` Constraint

[`ADD FOREIGN KEY … NOT VALID`](https://www.postgresql.org/docs/18/sql-altertable.html) takes `SHARE ROW EXCLUSIVE` on both the referencing and referenced tables for the catalog update

[`VALIDATE CONSTRAINT`](https://www.postgresql.org/docs/18/sql-altertable.html) takes `SHARE UPDATE EXCLUSIVE` on the referencing table and `ROW SHARE` on the referenced table

New rows are checked from the moment the constraint is added

Two rules follow from the lock profile:

- One FK per transaction — acquiring `SHARE ROW EXCLUSIVE` on two tables in one transaction can cause a deadlock
- `ADD … NOT VALID` and `VALIDATE CONSTRAINT` must run in separate transactions — otherwise the validation scan inherits the `SHARE ROW EXCLUSIVE` held since `NOT VALID`, blocking writes on both tables for the full scan

1. Deploy a code change to stop writing rows that would violate the FK (e.g. orphan rows)
2. Register the FK without validation:

   ```sql
   ALTER TABLE t1 ADD CONSTRAINT t1_c_fk FOREIGN KEY (c) REFERENCES t2 (id) NOT VALID;
   ```
3. Clean up violating rows in batches, outside any transaction
4. Validate the existing rows:

   ```sql
   ALTER TABLE t1 VALIDATE CONSTRAINT t1_c_fk;
   ```

For partitioned tables, validate each partition first, then attach to the parent

### Drop a `NOT NULL` Constraint

Takes a short `ACCESS EXCLUSIVE`, without a table rewrite

If the constraint was added as a named table constraint, drop it by name:

```sql
ALTER TABLE t DROP CONSTRAINT t_c;
```

Or via the column-level form, which works regardless of how the constraint was added:

```sql
ALTER TABLE t ALTER COLUMN c DROP NOT NULL;
```

For partitioned tables, drop on the parent — partitions inherit it

### Add a `UNIQUE` Constraint

1. Create a new index:

   Takes `SHARE UPDATE EXCLUSIVE`

   ```sql
   CREATE UNIQUE INDEX CONCURRENTLY t_c_uniq ON t (c);
   ```

   Note that `CREATE UNIQUE INDEX CONCURRENTLY` fails if any duplicates exist at build time, leaving an `INVALID` index behind
2. Deploy a code change that rejects any new duplicate writes
3. Convert an index to the constraint:

   Takes a short `ACCESS EXCLUSIVE`

   ```sql
   ALTER TABLE t ADD CONSTRAINT t_c_uniq UNIQUE USING INDEX t_c_uniq;
   ```

## Enum Types

### Add or Rename a Value

[`ALTER TYPE`](https://www.postgresql.org/docs/18/sql-altertype.html) does not lock tables that reference the enum:

```sql
ALTER TYPE my_enum ADD VALUE 'x';
ALTER TYPE my_enum RENAME VALUE 'a' TO 'b';
```

Note that `ADD VALUE` cannot run in the same transaction that later references the new value

# Backfills

A backfill loops a batched write until every row is done. They run between the expand MR's deploy and the contract MR.

```sql
with batch as (
   select <pk_cols> 
   from <schema>.<table>
   where (<pk_cols>) > ($1, $2, ...) -- Tuple comparison. Omitted for the first batch.
   order by <pk_cols> asc
   limit $3
   for no key update
),
updated as (
   update <schema>.<table> t 
   set <column> = <expression>
   from batch
   where (t.<pk_cols>) = (batch.<pk_cols>) and t.<column> is null
)
select <pk_cols>
from batch
order by <pk_cols> desc
limit 1;
```

> [!NOTE]
> A simpler pattern without a cursor slows down backfills over time.
>
> See [Postgres Job Queues & Failure By MVCC](https://brandur.org/postgres-queues) and [LIMIT-only loop-based batching strategy](https://gitlab.com/gitlab-org/gitlab/-/work_items/544662).
