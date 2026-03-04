---
title: 'Scripts for Data Fixes: Lessons Learned'
date: '2026-03-04'
description: Practical patterns for writing datafix scripts — rate limiting, batching, checkpoints, safe NOT NULL migrations, and more. Based on real experience with PostgreSQL at scale.
draft: false
url: /scripts/
---

At some point, every engineer runs into the need to do a datafix — patch up data after an incident, backfill missing values, or reconcile broken state across services. This post covers what I've learned writing these scripts, and some practical tips to make the process less painful for you and your team.

**Context:**

- The database is under production load — uncontrolled extra traffic causes 5xx errors and triggers monitoring alerts
- Tables have tens of millions of rows, with indexes
- Production DB access is typically restricted and granted for short windows
- Tables are connected via CDC (Change Data Capture) with per-transaction row limits
- Bulk updates may take days, even running overnight
- Stack: PostgreSQL, Go

**Common scenarios:**

- Reconciling a read model against the aggregate owner service
- Backfilling or enriching data
- Fixing data corrupted by a service bug
- Replaying idempotent domain events to restore consistency
- One-off workflows that aren't worth automating yet
- Fixing cross-system consistency violations
- Aggregating data from multiple services
- Large-scale DB migrations
- Adding a NOT NULL constraint to an existing column
- Migrating data to cold storage

If it's just a handful of rows and you have production DB access, a manual query is often the fastest path.

## Always wrap queries in a transaction

Before running any manual update, wrap it in a transaction: add `begin` at the top and leave `--commit` commented out. Run the `begin` and the update first, verify the results look right, then uncomment and run `commit`.

```sql
begin;
update transactions set status = 3
where transaction_id = 1679036763
--select status, transaction_id from transactions where transaction_id = 1679036763
--commit;
```

This technique has saved me from mistakes more than once — and makes for a much better night's sleep.

## Leave a paper trail

Every DB fix should be logged in your team's task tracker so you can later answer questions like "what happened to order #1679036763?" If the request came from support, open a ticket and paste the SQL along with the output into the description or a comment. Make this a team habit — it will save everyone a lot of pain down the road.

## Simple scripts

Anything beyond a handful of SQL calls is better off as a script — a small program that connects to the DB, runs queries in a loop, and exits cleanly. My examples are Go-centric, but the patterns apply to any language.

If a script is tightly coupled to one service and its database, keep it in that service's repository. It will stay in sync as the service evolves, and it's easy to find.

```
project-root/
├── cmd
│   └── service
│       └── main.go
├── internal
│   └── ...
├── scripts
│   ├── utils
│   │   ├── db.go
│   │   └── config.go
│   └── reconciliation
│       ├── .env.example
│       ├── .gitignore
│       ├── handler.go
│       ├── main.go
│       └── README.md
├── swagger.yaml
└── tools
```

Always include a `README.md` with setup and run instructions.

- `main.go` handles the boilerplate: wiring dependencies, reading config, opening the DB connection, creating the `Handler`, and calling its methods.
- `handler.go` contains the actual script logic.

Tips:

- **Backup** — Think about whether you need to snapshot the data before modifying it.
- **Caching** — If the script calls an external service via RPC and those calls are idempotent, cache the responses to a JSON file on disk. This makes the feedback loop much faster when iterating locally against a staging DB.
- **Rate limiting (DB)** — Throttle your DB queries. The `uber-go/ratelimit` library implements the leaky bucket algorithm and lets you set a comfortable RPM that won't spike DB load.
- **Rate limiting (RPC)** — Same idea for external service calls. Add a rate limiter per client, each with its own configurable limit.
- **Retries** — Add retries to network calls. A transient timeout shouldn't kill a long-running script.

## Bulk updates on a loaded database

Say you need to update 20 million rows. The DB is under heavy traffic — even well-optimized queries during the day push error rates up. So you run the script at night, when organic load drops. Reliability becomes critical: add a rate limiter for DB queries and retries for network calls, so a stray timeout doesn't abort the entire job.

### Scanning by sequential ID

If the table has an auto-incrementing primary key (e.g. `eid bigserial`), you can iterate by range — passing `minEID` and `maxEID` to process rows in slices of, say, 10,000. The B-tree index on the primary key makes these range scans efficient.

Benchmark on a small slice first (1,000 rows), then size your nightly window so the script finishes before morning traffic returns.

Batch your updates for better throughput — one SQL statement per N rows. Make the batch size configurable. In Go, `lo.Chunk` from [samber/lo](https://github.com/samber/lo?tab=readme-ov-file#chunk) is handy for this.

The script advances from `minEID` to `maxEID`. Since the data is ordered by key, if it crashes you can resume from where it stopped. Log the last successfully processed EID — to stderr or a temp file. Even better, implement checkpoint logic: on startup, read the checkpoint file and pick up from the last successful position.

This approach means you never need to fetch IDs into the application first — less network traffic, better performance. However, it doesn't always apply:

- No indexed sequential ID exists — e.g. a composite key like `(project, order, item)`
- The target rows are sparse — scanning the full `eid` range would be wasteful

In those cases, fetch the IDs with a `SELECT` first, chunk them in the application, and process from there. Make sure the query hits an index and returns sorted results.

### Filtering unprocessed rows

When filling a nullable column, you can identify unprocessed rows simply by filtering on `IS NULL`:

```sql
select project_id, order_id
from authorizations
where 1=1
  and provider_id is null  --  (1)
  and project_id in ('auto', 'goods', 'used', 'rent') -- all possible project ids
  and order_id >= $1 and order_id < $2
limit $3;
```

When there's no such marker — for example, you're changing `provider_id` from one non-null value to another, or you need to visit each row exactly once — you'll need a cursor or `LIMIT`/`OFFSET`. Watch out for a subtle trap: combining `LIMIT`/`OFFSET` pagination with a `WHERE ... IS NULL` filter will silently skip rows as the result set shifts underneath you.

### Adding NOT NULL safely

**Real-world example:** the `authorizations` table had a nullable `provider_id` column. At some point the team decided it should be required — new records started getting it populated. The challenge was backfilling tens of millions of historical rows, where the correct `provider_id` had to be derived in application code based on the order's creation year, project, and other attributes.

Once the backfill was done, naively running `SET NOT NULL` would have caused an outage — PostgreSQL acquires an `ACCESS EXCLUSIVE` lock and scans the entire table:

```sql
-- DON'T do this on a large, live table
alter table authorizations
alter column provider_id set not NULL;
```

The safe approach:

1. Backfill all rows using the script.
2. Add a `NOT VALID` check constraint — this is instant, no table scan:

```sql
ALTER TABLE authorizations
  ADD CONSTRAINT authorizations_provider_id_check_notnull
  CHECK (provider_id IS NOT NULL) NOT VALID;
```

3. Validate the constraint during off-hours. This scans all rows but only takes a `SHARE UPDATE EXCLUSIVE` lock (reads and writes still work). If any null remains, the command fails — fix the gaps and retry:

```sql
ALTER TABLE public.authorizations
  VALIDATE CONSTRAINT authorizations_provider_id_check_notnull;
```

4. Once the constraint is valid, `SET NOT NULL` becomes instant — PostgreSQL recognizes the existing check constraint and skips the scan. From the [PostgreSQL 12 release notes](https://www.postgresql.org/docs/release/12.0/):

> *Allow ALTER TABLE ... SET NOT NULL to avoid unnecessary table scans (Sergei Kornilov). This can be optimized when a table has a check constraint that is sufficient to prove that the column contains no nulls.*

```sql
ALTER TABLE public.authorizations
  ALTER COLUMN provider_id SET NOT NULL;
```

Tips:

- **Batching** — When enriching rows with data from an external service, batch your DB writes into a single statement per N rows. Make the batch size configurable.

## Fetch, transform, persist

Sometimes you need to aggregate data from multiple services, run transformations, and write the result to a database. For example: migrating state from a legacy B2C payout system to a new one with a completely different data model, across hundreds of thousands of customers. For each customer, you'd fetch financial reports for their activity periods, run calculations, and insert the results into the new service's DB.

Make the script idempotent — safe to run multiple times without corrupting data. A good pattern is to split the work into stages:

- **Stage 1: Fetch and compute.** Pull data from source services, run your business logic, and write intermediate results to files on disk. Cache network responses if the data is immutable — this dramatically speeds up iteration when debugging against staging. Add every integrity check you can think of: validate field values, verify totals, assert invariants.

- **Stage 2: Load.** Read the computed files and insert into the target DB. This stage ends up much simpler — almost no logic, minimal testing. Consider building in a cleanup path: reserve a range of IDs (e.g. for `report_id`) that are safe to wipe between test runs. If the new service isn't live yet, you can simply truncate the target tables.

Splitting the pipeline this way gives you a clear boundary: all the complexity lives in stage 1, where you can iterate quickly with cached data and thorough validation. Stage 2 is a straightforward load that's easy to verify and safe to re-run.

## Final note: scripts are a stepping stone

Every technique in this article lives at stage 2 of a natural maturity curve that most teams go through:

1. Manual investigation and one-off SQL fixes
2. **Reusable scripts** with runbooks and on-call documentation
3. Dashboards, metrics, trend tracking
4. Dedicated ownership — an engineer, then a manager, then a team
5. Architectural improvements — clearer service boundaries, admin tooling, auto-remediation
6. Root causes eliminated — the problematic integration replaced, the redesign shipped

Scripts are a healthy and practical response when the team is moving fast and can't invest in full automation yet. But if you find yourself running the same script every week, that's a signal — not a failure, but an opportunity. The goal is to keep climbing the ladder: each step reduces toil, improves reliability, and frees up engineering time for work that matters.

---

Denis Khranovskii, 2026

tags: datafix, data migration, scripts
