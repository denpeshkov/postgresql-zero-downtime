# Expand-Contract Pattern

## The Two Coexisting Schemas

There is only **one physical schema** — the real tables — but the database presents **two logical schemas** at the same time:

- **The old logical schema** — what the previous application version expects: the original column names, types, and constraints
- **The new logical schema** — what the next application version expects: the renamed column, the new type, the added constraint, etc.

Old pods talk to the old logical schema, new pods talk to the new one, and the database (or application) keeps the data they see consistent

Typically implemented with **versioned views** backed by the same underlying tables, each exposing the data shape the corresponding application version expects

A pod selects its logical schema once at connection time (e.g. via `search_path`) and is unaffected from the rollout from then on

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

After Contract, only the new shape remains. The transition is completea

# Migrations Guideline

## Locking

DDL that takes `ACCESS EXCLUSIVE` is "fast" in isolation but can stall the database if it queues behind a long-running query:
while the migration waits, every subsequent reader queues _behind_ it

Two defenses, applied to every DDL session:

1. Query [`pg_locks`][pg_locks] for long-running blockers on the target table; pause and retry until the queue is short
2. Bound the wait with `lock_timeout`:

    ```sql
    BEGIN;
    SET LOCAL lock_timeout = '2s';   -- fail if not acquired in 2 s
    ALTER TABLE t ADD COLUMN c text;
    COMMIT;
    ```

Operations that take a _weaker_ lock — `CREATE INDEX CONCURRENTLY`, `ALTER TABLE … VALIDATE CONSTRAINT` — wait without blocking writers and don't need this protection

[pg_locks]: https://www.postgresql.org/docs/18/view-pg-locks.html

## Tables

### Create a Table

Single migration. [`CREATE TABLE`][create_table] does not lock existing tables.
If the table has multiple foreign keys, split them across migrations:
each `ADD FOREIGN KEY` locks the referenced table, and holding several locks at once raises deadlock risk

1. Create the table with indexes, no foreign keys
2. Add the first foreign key (`NOT VALID` + `VALIDATE`)
3. Add the next foreign key

[create_table]: https://www.postgresql.org/docs/18/sql-createtable.html

### Rename a Table

Two deployments. [`ALTER TABLE … RENAME`][alter_table] takes `ACCESS EXCLUSIVE` very briefly, but client code holding the old name
keeps using it until the rolling deploy completes

1. **Deploy N**: rename the table and create a view under the old name in the same transaction.
   Reads, inserts, updates, and deletes against the view continue to work — see the [updatable views caveats][updatable_views]

    ```sql
    BEGIN;
    ALTER TABLE t_old RENAME TO t_new;
    CREATE VIEW t_old AS SELECT * FROM t_new;
    COMMIT;
    ```

2. **Deploy N+1** (post-deploy): drop the view once every client is on the new name

    ```sql
    DROP VIEW t_old;
    ```

[alter_table]: https://www.postgresql.org/docs/18/sql-altertable.html
[updatable_views]: https://www.postgresql.org/docs/18/sql-createview.html#SQL-CREATEVIEW-UPDATABLE-VIEWS

### Drop a Table

Two deployments. `DROP TABLE` takes `ACCESS EXCLUSIVE`

1. **Deploy N**: remove every reference to the table from the code
2. **Deploy N+1** (post-deploy): `DROP TABLE t;`

Sanity-check via [`pg_stat_user_tables`][pg_stat_user_tables] that the table is no longer being read before scheduling Deploy N+1

[pg_stat_user_tables]: https://www.postgresql.org/docs/18/monitoring-stats.html#MONITORING-PG-STAT-USER-TABLES-VIEW

## Columns

### Add a Column

Single migration. `ALTER TABLE … ADD COLUMN` takes a brief metadata-only `ACCESS EXCLUSIVE` lock.
PostgreSQL stores defaults in the catalog, so adding a column with a constant default does not rewrite the table

- **Nullable, no default** -> direct
- **With a constant default** -> direct (no rewrite)
- **`NOT NULL` with no default** -> not safe in one step. Either add a default, or add the column nullable and follow [Add NOT NULL](#add-not-null)

### Drop a Column

Two deployments. `DROP COLUMN` takes `ACCESS EXCLUSIVE`

1. **Deploy N**: remove every reference to the column from the code.
   The column stays in the database until every old service instance is gone — old binaries still issue queries that mention it
2. **Deploy N+1**:
    1. Find dependents in [`pg_depend`][pg_depend] catalog
    2. Drop dependent indexes with `DROP INDEX CONCURRENTLY` first; otherwise they would be dropped along with the column under `ACCESS EXCLUSIVE`
    3. If a view references the column, recreate the view without it
    4. `ALTER TABLE t DROP COLUMN c;`

Sanity-check via `pg_stat_user_tables` that the column is no longer being read before scheduling Deploy N+1

[pg_depend]: https://www.postgresql.org/docs/18/catalog-pg-depend.html

### Rename a Column

Two deployments + backfill. Both names must coexist while the rolling
deploy is in progress, because old code still uses the old name and new code uses the new one

1. **Expand (Deploy N)**:
    1. Add the new column
    2. Dual-write to both columns via a `BEFORE INSERT/UPDATE` trigger (or in application code)
2. **Backfill**: Copy existing data from old column to new column in batches. Verify completion
3. **Contract (Deploy N+1)**:
    1. Switch the app to read/write the new column
    2. Drop the trigger and the old column in a post-deploy migration

If a view references the column, recreate it pointing at the new column during Expand phase

### Change a Column's Type

One migration if the change is catalog-only; two deployments plus a backfill otherwise.
Either way `ALTER COLUMN … TYPE` takes `ACCESS EXCLUSIVE`, but a rewrite holds it for the full scan

Verify if changes are catalog-only on a thin-clone by checking that [`pg_class.relfilenode`][pg_class] doesn't change

For changes that rewrite the column:

1. Add a new column of the target type
2. Dual-write to both columns via a `BEFORE INSERT/UPDATE` trigger (or in application code)
3. Backfill the new column in batches. Validate the cast on a clone first — a failed cast mid-backfill leaves the column half-migrated
4. Swap names in a single transaction with a short [`LOCK TABLE`][lock_table]
5. Drop the old column in a post-deploy migration

[pg_class]: https://www.postgresql.org/docs/18/catalog-pg-class.html
[lock_table]: https://www.postgresql.org/docs/18/sql-lock.html

### Add `NOT NULL`

Two deployments. PostgreSQL 18 supports a [native `NOT NULL … NOT VALID` constraint][pg18_release], which avoids
the long table scan that the pre-18 `SET NOT NULL` form required

The flow is:

1. **Deploy N**:
    1. Update code so it never writes `NULL` to the column
    2. Backfill `NULL` rows in batches in a post-deploy data migration
2. **Deploy N+1**: register and validate the constraint without a long exclusive lock:

    ```sql
    ALTER TABLE t ADD CONSTRAINT t_c_not_null NOT NULL c NOT VALID; -- Brief ACCESS EXCLUSIVE: catalog-only, no scan
    ALTER TABLE t VALIDATE CONSTRAINT t_c_not_null;  -- SHARE UPDATE EXCLUSIVE: scans existing rows; reads and writes continue concurrently
    ```

After `VALIDATE` the column behaves as if it were declared `NOT NULL` at column level — the optimizer treats it the same.
There is no need for a follow-up `SET NOT NULL` step on PG 18

[pg18_release]: https://www.postgresql.org/docs/18/release-18.html

### Drop `NOT NULL`

Single migration. Brief metadata-only `ACCESS EXCLUSIVE`, no rewrite

```sql
ALTER TABLE t ALTER COLUMN c DROP NOT NULL;
```

For partitioned tables, drop on the parent — partitions inherit it

## Indexes

### Create an Index

Single migration. [`CREATE INDEX CONCURRENTLY`][create_index] takes `SHARE UPDATE EXCLUSIVE` and does not block reads or writes

It cannot run inside a transaction and requires two table scans, so it's slower than a plain `CREATE INDEX`.
It also waits for in-flight transactions on the table to complete before finishing — it can stall behind a long query

```sql
CREATE INDEX CONCURRENTLY t_c_idx ON t (c);
```

If the build fails, the database leaves an `INVALID` index. Drop it before retrying:

```sql
DROP INDEX CONCURRENTLY IF EXISTS t_c_idx;
```

[create_index]: https://www.postgresql.org/docs/18/sql-createindex.html

### Drop an Index

Single migration. `DROP INDEX CONCURRENTLY` takes `SHARE UPDATE EXCLUSIVE`.

```sql
DROP INDEX CONCURRENTLY t_c_idx;
```

Cannot drop an index backing a `PRIMARY KEY` or `UNIQUE` constraint — drop the constraint instead, which removes the index

### Reindex an Index

Single migration. Same `SHARE UPDATE EXCLUSIVE` lock and caveats as `CREATE INDEX CONCURRENTLY`

```sql
REINDEX INDEX CONCURRENTLY t_c_idx;
```

## Constraints

### Add a Foreign-Key or `CHECK` Constraint

Single migration, two-step.
[`NOT VALID`][not_valid] registers the constraint without scanning existing rows under a brief `ACCESS EXCLUSIVE` (or `SHARE ROW EXCLUSIVE` on both sides for foreign keys);
[`VALIDATE CONSTRAINT`][validate] scans them later under `SHARE UPDATE EXCLUSIVE`, allowing concurrent reads and writes (plus `ROW SHARE` on the referenced table for foreign keys).
New rows are checked from the moment the constraint is added

```sql
-- Foreign key (t1.c -> t2.id)
ALTER TABLE t1 ADD CONSTRAINT t1_c_fk FOREIGN KEY (c) REFERENCES t2 (id) NOT VALID;
ALTER TABLE t1 VALIDATE CONSTRAINT t1_c_fk;

-- CHECK
ALTER TABLE t ADD CONSTRAINT t_c_check CHECK (c > 0) NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT t_c_check;
```

For partitioned tables, validate each partition first, then attach to the parent

(`NOT NULL` follows the same pattern but is documented separately under [Columns -> Add NOT NULL](#add-not-null) for the deployment context)

[not_valid]: https://www.postgresql.org/docs/18/sql-altertable.html
[validate]: https://www.postgresql.org/docs/18/sql-altertable.html

### Add a `UNIQUE` Constraint

Single migration, two-step. The plain form `ADD CONSTRAINT … UNIQUE (col)` builds the backing index under `ACCESS EXCLUSIVE`.
Build the index concurrently first (`SHARE UPDATE EXCLUSIVE`), then attach it to the constraint with a brief metadata-only `ACCESS EXCLUSIVE`:

```sql
CREATE UNIQUE INDEX CONCURRENTLY t_c_uniq ON t (c);
ALTER TABLE t ADD CONSTRAINT t_c_uniq UNIQUE USING INDEX t_c_uniq;
```

The `ADD CONSTRAINT` step is a fast metadata-only operation

## Enum Types

### Add or Rename a Value

Single migration. [`ALTER TYPE`][alter_type] does not lock tables that reference the enum:

```sql
ALTER TYPE my_enum ADD VALUE 'x';
ALTER TYPE my_enum RENAME VALUE 'a' TO 'b';
```

`ADD VALUE` cannot run in the same transaction that later references the new value

[alter_type]: https://www.postgresql.org/docs/18/sql-altertype.html
