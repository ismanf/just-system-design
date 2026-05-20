# Database Migrations at Scale

> **TL;DR** — A **database migration** is a controlled change to a schema or data: adding a column, dropping a table, renaming a field, backfilling values. At small scale, your migration tool (Alembic, Flyway, Liquibase, Sqitch, Rails, Prisma) handles it. At scale, every migration must be **online**, **safe to roll back**, **deployed independently of code**, and aware of **replicas, read paths, and ongoing traffic**. The recipe: **expand → migrate → contract**, with every step backward-compatible. Big-bang renames die badly. The single most important habit: never deploy code and schema change in lockstep.

---

## 1. What "Migration" Means

Three categories:
- **DDL migrations** — `CREATE TABLE`, `ADD COLUMN`, `DROP INDEX`. Affect schema.
- **Data migrations** — `UPDATE ... SET new_col = expr` across millions of rows. Affect data.
- **Software-coupled migrations** — schema + code change together. Almost always the dangerous kind.

A safe practice treats them all under the same banner: **versioned, reviewed, idempotent, repeatable**.

---

## 2. Why It Gets Hard at Scale

A 10-row table accepts any change instantly. A **1-billion-row table** doesn't.

- `ADD COLUMN x int DEFAULT 7` may rewrite the entire table (old Postgres). Hours of downtime.
- `ALTER COLUMN` can take an exclusive lock for the duration of the rewrite.
- `CREATE INDEX` without `CONCURRENTLY` locks writes.
- `DROP COLUMN` is fast — but every replica must catch up.
- A naive `UPDATE` of a billion rows holds locks and bloats the table.
- A misordered code/schema deploy crashes the running app.

You have to **think like a distributed system** about your schema, even on a single-node DB.

---

## 3. Backwards Compatibility: The Golden Rule

> The application must run correctly with both the old schema and the new schema for the entire deployment window.

If you can promise that, deploys and migrations are independent and you can roll back without coordinating both.

If you can't, you have a **lockstep deploy** — and any rollback becomes a multi-step recovery.

Concrete consequences:
- **Adding a column?** Make it nullable or have a default. Old code never references it.
- **Removing a column?** First remove it from code, deploy, *then* drop it.
- **Renaming a column?** Add the new column → write to both → backfill → switch reads → stop writing the old → drop old. (See expand-contract below.)
- **Changing a type?** Add new column, dual-write, backfill, switch reads, drop old.

The migration is split into safe atomic steps that *each* are backwards-compatible.

---

## 4. The Expand–Migrate–Contract Pattern

The canonical playbook for any non-trivial change.

```
1. EXPAND   add the new shape next to the old.
            both shapes coexist; code reads/writes either.

2. MIGRATE  backfill data into the new shape (batched, online).
            switch reads to the new shape gradually (feature flag).
            verify counts, correctness.

3. CONTRACT  stop writing the old shape (code change).
             after a soak period, drop the old shape.
```

Every step is independently deployable and rollback-safe.

### Worked example: renaming `email` → `email_address`

1. **Add column** `email_address`. Deploy.
2. **Code dual-writes** to both columns. Deploy.
3. **Backfill** old rows: `UPDATE users SET email_address = email WHERE email_address IS NULL` (in batches!).
4. **Verify** every row has both populated.
5. **Code switches reads** to `email_address`. Deploy.
6. **Stop writing** `email`. Deploy.
7. **Drop** column `email` after a soak (days or weeks). Deploy.

Seven steps, each one a tiny safe change. *That's* a scale-safe migration.

---

## 5. Online Schema Changes

Modern DB engines support more online operations than they used to:

### Postgres
- **`ADD COLUMN` with DEFAULT** is metadata-only since PG11 (was a full rewrite in PG10).
- **`CREATE INDEX CONCURRENTLY`** builds an index without blocking writes (and `DROP INDEX CONCURRENTLY` for the reverse).
- **`ALTER TABLE ... VALIDATE CONSTRAINT`** lets you add a constraint as `NOT VALID` then validate online.
- **Generated columns**, **partitioning**, **NULL/NOT NULL with `CHECK NOT VALID`** patterns let you avoid table rewrites.

### MySQL
- **InnoDB online DDL** (`ALGORITHM=INPLACE`) handles many changes online.
- **gh-ost** (GitHub's online migration tool) creates a shadow table, copies data in chunks, and atomically swaps.
- **pt-online-schema-change** (Percona) does similar.
- **Vitess** has built-in **VReplication**-based online migrations.

### Cloud-managed (Aurora, AlloyDB, Cloud SQL)
- Similar to underlying engine; sometimes additional restrictions (no superuser).

Always check **what's supported online** for your engine and version *before* you start.

---

## 6. Data Migrations: Batch It

A 1B-row `UPDATE` in one statement holds locks, bloats the table, and pegs the DB. Always **batch** by primary key or time:

```sql
-- Postgres-style batched update
DO $$
DECLARE
  last_id bigint := 0;
  batch  int    := 5000;
BEGIN
  LOOP
    UPDATE users
       SET email_address = email
     WHERE id BETWEEN last_id + 1 AND last_id + batch
       AND email_address IS NULL;
    EXIT WHEN NOT FOUND;
    PERFORM pg_sleep(0.05); -- yield
    last_id := last_id + batch;
  END LOOP;
END $$;
```

Patterns:
- **Small batches** (1k–10k rows).
- **Commit each batch** to avoid one giant transaction.
- **Throttle** between batches (`pg_sleep`, `time.sleep`).
- **Resumable** — record progress so a restart picks up where it left off.
- **Idempotent** — a re-run shouldn't double-apply.
- **Off-peak** — bigger batches at 3 AM, smaller during the day.

For very large jobs, run as a **background worker** with a job queue or a dedicated tool (Bytebase, Liquibase Pro, Skeema, gh-ost).

---

## 7. Index Creation Without Downtime

In Postgres:
```sql
CREATE INDEX CONCURRENTLY users_email_idx ON users (email);
```
- No exclusive lock; takes longer.
- Can fail mid-build; resulting index is marked `INVALID` — drop and retry.
- Cannot run inside a transaction.

In MySQL:
- InnoDB does most index ops online.
- `gh-ost` for the trickier cases.

Always build indexes online on production. Even small tables under heavy write load can suffer if you take an exclusive lock.

---

## 8. Dropping Columns and Tables

- **Drop column** is fast (Postgres marks the column dead; physical removal happens on `VACUUM FULL`).
- **Drop table** is fast.
- The risk is **app code still references it**. Use the contract phase: ensure no code paths still touch the column, soak in production, *then* drop.

For belt-and-suspenders: rename it first to `_deprecated_email` and watch for errors / metrics for a week. If silence, drop.

---

## 9. Adding NOT NULL and Constraints

A new `NOT NULL` constraint will scan and validate the whole table → exclusive lock → outage on a big table.

Safe approach (Postgres):
1. `ALTER TABLE users ADD CONSTRAINT users_email_not_null CHECK (email IS NOT NULL) NOT VALID;` — instant, doesn't validate existing rows but enforces on new ones.
2. Backfill any `NULL`s.
3. `ALTER TABLE users VALIDATE CONSTRAINT users_email_not_null;` — scans without blocking writes.
4. (Optional) Later, add the actual `NOT NULL` column attribute when downtime is feasible.

Same idea for foreign keys (`NOT VALID` + `VALIDATE`).

---

## 10. Migrations + Replicas

In a primary-replica setup, DDL replicates through the WAL. Long DDL on the primary delays replicas' apply. Consequences:
- **Replica lag balloons** during a long migration.
- **Read traffic on replicas degrades.**
- **Failover during the migration** is dangerous.

Mitigation:
- Schedule migrations during low traffic.
- Watch replica lag explicitly.
- Use online DDL whenever possible (no exclusive lock = small lag).
- For huge data backfills, run them only on the primary; replicas catch up via WAL.

---

## 11. Migrations + Sharded Systems

Each shard runs its own copy of the migration. Coordination matters:
- All shards must end up at the same schema version.
- A long migration on one shard slows the rest of the platform's deploys.
- Some tools (Vitess, Citus) coordinate migrations centrally.

Patterns:
- **Idempotent** migration scripts (re-running is safe).
- **Versioned** migration metadata per shard.
- **Health checks** to confirm migration before claiming completion.
- **Rolling deploys** of code that's compatible with both old and new schemas across shards.

---

## 12. Tools

### Versioning / orchestration
- **Flyway** (JVM) — versioned SQL files; simple, ubiquitous.
- **Liquibase** — XML/YAML/SQL with rollback hooks.
- **Alembic** (Python / SQLAlchemy).
- **Sqitch** — DAG-based migrations with dependency tracking.
- **Goose / dbmate / golang-migrate** — Go-flavored.
- **Prisma Migrate / Knex / TypeORM / Drizzle Kit** — TS / JS.
- **Active Record migrations** (Rails).
- **Bytebase / Atlas / Hasura / dbt** — more declarative / GitOps-flavored.

### Online migration tools
- **gh-ost** (GitHub) — MySQL.
- **pt-online-schema-change** (Percona) — MySQL.
- **Vitess online DDL** — sharded MySQL.
- **pg_repack** — Postgres bloat / re-pack online.
- **pgroll** — Postgres expand-contract migrations as a tool.
- **Reshape** — Postgres zero-downtime migrations.

### Schema diff / linting
- **Atlas** by Ariga — schema-as-code, drift detection, lint, plan.
- **Squawk** — Postgres migration linter (catches "table rewrite" before deploy).
- **OpenAPI / dbt** for declarative analytics models.

---

## 13. Migration Workflow That Actually Works

```
1. Author the migration in a PR. Include rollback.
2. Lint locally (Squawk / Atlas).
3. Run in dev / staging — full-size if possible.
4. Code reviewer checks for:
     - blocking DDL on big tables
     - mass UPDATEs without batching
     - one-shot transactional UPDATEs
     - missing NOT VALID / VALIDATE
     - cross-table FK introduction with full scan
5. Merge → CI runs migration in staging again with prod-sized data dump.
6. Deploy to production via the orchestration tool.
7. Verify with health checks; watch replica lag.
8. Soak before contracting.
9. Always have a rollback plan documented in the PR.
```

Treat migrations like production code — because they are.

---

## 14. Zero-Downtime Deploy Mechanics

The choreography:
```
Code v1 (knows only old schema)
   ↓ deploy
Migration step A (expand: add new column, both NULL OK)
   ↓
Code v2 (knows both schemas, prefers new on writes, reads from old)
   ↓ deploy
Migration step B (backfill new column in batches)
   ↓
Code v3 (reads from new, dual-writes)
   ↓ deploy
Migration step C (stop writing old, drop old)
   ↓
Code v4 (clean — only new schema)
```

Each arrow is *independent*. Rollback at any step doesn't break the system.

For really sensitive systems, gate every code change behind a **feature flag** so behavior switches without redeploy.

---

## 15. Common Mistakes

- **Bundling code + schema change** in one deploy. Rollback nightmare.
- **`ADD COLUMN NOT NULL DEFAULT X`** on a big table (old engines rewrite).
- **One massive `UPDATE`** — locks, bloat, replica lag, abort risk.
- **`CREATE INDEX`** without `CONCURRENTLY` on a hot table.
- **Renaming columns in-place** instead of expand-contract.
- **Migrations not idempotent** — re-running fails.
- **No rollback documented**.
- **DDL during peak** without staging dry-run.
- **Forgetting replicas** — primary fast, replicas slow.
- **Hand-running migrations** instead of using the migration tool.
- **Two PRs to the same migration version** — race conditions in CI.
- **"Just SSH and `psql`"** — invisible, unaudited, dangerous.

---

## 16. Cheat Card

```
GOAL  every step must be backwards-compatible.
       code and schema deploy independently.

PATTERN  EXPAND → MIGRATE → CONTRACT
  add new shape ▸ dual-write + backfill ▸ switch reads ▸ stop old writes ▸ drop old

ONLINE PRIMITIVES (Postgres)
  CREATE INDEX CONCURRENTLY
  ADD COLUMN with DEFAULT (metadata-only on PG11+)
  NOT VALID / VALIDATE CONSTRAINT
  generated columns

ONLINE TOOLS (MySQL)
  InnoDB online DDL · gh-ost · pt-online-schema-change · Vitess vDDL

DATA MIGRATIONS  always batch. small (1k–10k rows). throttle. resumable. idempotent.

NEVER
  one-shot UPDATE of millions of rows
  ALTER COLUMN that rewrites the table during peak
  rename-in-place without an expand-contract dance
  deploy code + schema in lockstep
  manual `psql` migrations

RULES
  always have a rollback path
  always test against prod-sized data in staging
  always watch replica lag during the change
  always run migrations through the tool, in version control
  always lint with Squawk / Atlas / linter of choice

REPLICAS / SHARDS
  expect lag during DDL on the primary
  sharded schema changes must be coordinated across shards
```

---

## 17. Resources

### Articles
- "Online migrations at scale" — Stripe engineering: <https://stripe.com/blog/online-migrations>
- "The Tortoise and the Hare: Online Schema Changes" — GitHub gh-ost blog: <https://github.blog/2016-08-01-gh-ost-github-s-online-migration-tool-for-mysql/>
- "Safe schema changes with Postgres" — Crunchy Data / pganalyze.
- "Postgres at scale: how Heroku does migrations" — Heroku.
- "Migrating tables with billions of rows" — many engineering blogs (Shopify, Stripe, Discord).
- "Common database schema changes to avoid" — Braintree blog.

### Documentation
- **Postgres ALTER TABLE** — <https://www.postgresql.org/docs/current/sql-altertable.html>
- **MySQL online DDL** — <https://dev.mysql.com/doc/refman/en/innodb-online-ddl.html>
- **gh-ost** — <https://github.com/github/gh-ost>
- **pt-online-schema-change** — <https://docs.percona.com/percona-toolkit/pt-online-schema-change.html>
- **Atlas** — <https://atlasgo.io/>
- **Flyway, Liquibase, Alembic** docs.
- **Squawk** — <https://squawkhq.com/>
- **pgroll** — <https://github.com/xataio/pgroll>
- **Reshape** — <https://github.com/fabianlindfors/reshape>

### Books
- *Database Reliability Engineering* — Laine Campbell, Charity Majors.
- *Refactoring Databases* — Scott W. Ambler, Pramod J. Sadalage.
- *PostgreSQL High Performance* — Greg Smith.

### Videos
- Hussein Nasser migration deep dives — <https://www.youtube.com/@hnasr>
- "gh-ost: GitHub's MySQL migration tool" — talk on YouTube.
- "Zero downtime migrations" — Stripe / Shopify conference talks.

### Adjacent reading
- [Relational Databases Deep Dive](./relational-databases.md)
- [MVCC](./mvcc.md)
- [Indexing](./indexing.md)
- [Replication](./replication.md)
- [Read Replicas](./read-replicas.md)
- [Sharding & Partitioning](./sharding-partitioning.md)
- [Change Data Capture →](./cdc.md)

---

*Previous:* [← Connection Pooling](./connection-pooling.md)  |  *Next:* [Change Data Capture (CDC) →](./cdc.md)
