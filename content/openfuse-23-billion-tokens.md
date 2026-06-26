+++
title = "I Burned 23 Billion Tokens to \"Rewrite\" Langfuse"
date = 2026-06-26
description = "A migration retro: forking Langfuse to swap its analytics store from ClickHouse to GreptimeDB, run mostly by two AI agents. Most of the work was paying down the debt of one decision — make a single database both the source of truth and the analytics layer."

[taxonomies]
tags = ["GreptimeDB", "Observability", "AI", "Vibe Coding"]

[extra]
toc = true
+++

![I Burned 23 Billion Tokens to "Rewrite" Langfuse](/images/openfuse-23-billion-tokens-cover.webp)

TMA1 broke down this project's token spend by cwd (the project is a Langfuse fork). Claude Code came to `23,448,326,730`, about 23.4 billion.

That number is less scary than it looks. 98% of it is cache read (23 billion), prompt-caching hits that are basically free. The real non-cached input + output is only 123M. Codex adds a rough couple hundred million on top, again mostly cache.

Two AI agents (Claude Code + Codex), twenty-some billion tokens, and what they did was swap the analytics store under Langfuse from ClickHouse to GreptimeDB. The product, the API, the SDK: not a line touched, which is why "rewrite" gets the scare quotes. The fork is called Openfuse.

What follows isn't a release post. It's a migration retro. The whole thing came down to one bet: take the source of truth off S3 and let a single GreptimeDB instance be both the truth and the analytics layer. Most of the article is about paying down the debt that decision created. I ran the migration mostly with two agents driving, which is where the tokens went; but what got it to run wasn't the model, it was engineering discipline.

## Openfuse: running LLM engineering on a real observability database

[Openfuse](https://github.com/tma1-ai/openfuse/blob/main/README.md) is a hard fork of Langfuse (v3.184.1). The main line is one thing: move the analytics store from ClickHouse to GreptimeDB. Postgres stays put: it still holds users / projects / prompts / API keys and the rest of the app and config data, with schema and migrations lifted straight from upstream. Redis / BullMQ stay too. What moves is traces / observations / scores and all the analytics data behind the dashboards; GreptimeDB now carries all of it.

![Openfuse dashboard](/images/openfuse-23-billion-tokens-dashboard.webp)

![Openfuse tracing view](/images/openfuse-23-billion-tokens-tracing.webp)

But "just swapping the store" is a simplification. To make it run on its own, the deployment shape and some object-storage paths changed too. Object storage is now optional and defaults to local files, and there's a standalone mode (web + worker bundled) you bring up in one command:
```bash
git clone https://github.com/tma1-ai/openfuse.git
cd openfuse
cp .env.quickstart.example .env                        # working dev defaults — no edits needed
docker compose -f docker-compose.standalone.yml up -d  # one app container + Postgres/Redis/GreptimeDB
```
For small-to-mid self-hosting, object storage is too heavy.

Why do this? LLM traces are observability data. High-cardinality, timestamped Wide Events are exactly GreptimeDB's data model. Running on one unified observability database instead of welding onto a single-purpose column store buys a few concrete things today:

- Start on a single node, scale to a cluster cleanly. One `openfuse-standalone` container runs the whole thing: web + worker in one process, talking to Postgres / Redis / GreptimeDB. Data lands on local disk or object storage, and the same engine scales from one node to a cluster as data grows. With persistence and object storage set up right, scaling the compute layer doesn't lose analytics data.
- Cheap long retention. Object-storage-native tiered storage plus a single SQL whole-database TTL keeps months-to-years of retention affordable. On ClickHouse-based Langfuse, configurable retention is an Enterprise feature.
- Ingestion doesn't hard-depend on object storage. You don't need S3 or MinIO to take data in; media uploads, OTel carriers, and eval blobs default to the local filesystem, and only the optional bulk export still needs S3.
- The write protocol is fully compatible: your Langfuse SDK code needs zero changes. Everything outside EE basically carried over, and the query side was validated against upstream on the same dataset.

Downsides exist. It's alpha. Some query semantics, approximate queries especially, don't fully match upstream because the underlying storage differs. And there's no in-place migration: no tool to move existing ClickHouse data into GreptimeDB, so it's a fresh install only. See [known limitations](https://github.com/tma1-ai/openfuse/blob/main/docs/known-limitations.md).

## Why the source of truth became raw_events

The thing that ate the most time was reconciling the semantics of the two stores. Before the pitfalls, one architectural decision, or the rest won't make sense: the source of truth moved from S3 to a single `raw_events` table in GreptimeDB.

![BEFORE (ClickHouse) vs AFTER (GreptimeDB) architecture](/images/openfuse-23-billion-tokens-architecture.webp)

How Langfuse worked before: an event comes in, gets uploaded to S3 (the event store, the source of truth), a message goes onto BullMQ, a worker downloads, parses, and enriches it (filling in prompt / model / cost), then a batching, async-flushing `ClickhouseWriter` writes it into ClickHouse. S3 is the truth; ClickHouse is a rebuildable projection.

Now: the API takes the event and appends the raw envelope straight into `raw_events` (`append_mode`); BullMQ only carries a lightweight reference. The worker reads the entity's full history from `raw_events`, enriches it, and writes the projection tables. The whole S3 layer is gone.

Two reasons. One, it's simpler: one fewer S3 event store, one fewer replay path, truth and analytics in one database, and a single system to back up, restore, and reprocess. Two, the write path has a different shape. ClickHouse dislikes high-frequency small writes: you batch, flush async, and lean on MergeTree background merges, which is why upstream puts S3 in front as a buffer with a batch writer ahead of it. GreptimeDB's write path (Mito Engine + WAL) is better suited to taking real-time appends directly. So I land raw events straight into `raw_events` instead of using S3 as the event store, and that whole tangle of lifecycle management goes away with it.

That decision also forced a string of semantics-alignment pitfalls.

## Reconciling the two stores' semantics

### last_non_null keeps the last write, not the largest event_ts

In ClickHouse, traces / observations / scores are all `ReplacingMergeTree(event_ts, is_deleted)`; you read with `FINAL`, taking the newest by the `event_ts` version column. GreptimeDB's counterpart is `merge_mode='last_non_null'`: for the same primary key + timestamp, it merges each field to its last non-NULL written value and backfills partial rows automatically. Looks one-to-one. The catch is in the word "last."

GreptimeDB's "last" goes by write sequence (a monotonically increasing per-Region write counter), not by the value of `event_ts`. I verified it in the PoC: write a logically newer event first, then a logically older one, and the older event that was written later wins. A stale event that arrives late just overwrites the newer value.

In a system that retries, replays out of order, and reprocesses concurrently, that's fatal. Two options: add field-level version guards in the worker (track the largest `event_ts` seen per field, skip older ones), or change the approach.

I changed the approach: full replay from `raw_events`. Instead of reading the projection and doing an incremental merge, the worker reads all of an entity's historical events, orders them deterministically (`event_ts` → create before update → `ingested_at` → `event_id`), merges from an empty object into a complete snapshot, and writes the whole row to the projection. Out-of-order goes away at the root: the rebuild depends only on the set of events, not their arrival order, so no version guard is needed. The cost is that every event triggers a full-history read and rebuild for that entity, so you have to watch the per-entity event count.

### The cost of dropping S3: raw_events TTL can't be a cost lever

Full rebuild has a prerequisite: complete history. That leads to the most concrete constraint after dropping the S3 SoT: `raw_events` TTL must be ≥ the projection's retention. If a create event expires first under TTL, the rebuild loses that entity's immutable fields / metadata / tags.

So in this architecture, TTL can't be a cost-shedding lever anymore (short of adding checkpoints / snapshots later). That's the price of putting truth and analytics in one database.

### Map / Array aren't first-class, so they get split into JSON + EAV side tables

ClickHouse's `Map(K,V)` + `sumMap` / `mapKeys` and `Array(String)` + `has()` have no first-class column equivalent in GreptimeDB. So each one gets mapped differently:

- Metadata goes to a JSON column plus an indexed EAV side table (`*_metadata`, inverted index on `key`, skipping index on `value`) that carries filtering and breakdowns.
- Tags go to a JSON array plus an append-only `*_tags`.
- Common usage / cost keys flatten into explicit decimal columns, so `sumMap` degrades to `SUM(column)`. The long tail spills into JSON.

This one rippled the furthest: enrich, converter, filter DSL, and query builder all had to follow. The JSON v2 GreptimeDB is currently designing could borrow more from CH; that Map / function setup really is nicer to write queries against.

### A few small but annoying ones

- Keywords. `id`, `name`, `value`, `key`, `type`, `level`, `timestamp` are all keywords in GreptimeDB, so column names have to be quoted consistently.
- UI edits nearly got dropped silently. tRPC mutations like bookmarking, the public toggle, and manual scoring used to write ClickHouse directly; my write path only covered ingestion at first, so after the cutover those edits would land in a database about to be deleted and never reach the projection. The fix: append a synthetic `*-create` event to `raw_events` (keeping the SoT complete and replayable) plus write the projection directly (read-after-write, visible immediately).
- Deletes. Deleting an entity isn't a real delete; it appends a tombstone to `raw_events` and drops the current projection / EAV. Replay then rebuilds it as soft-deleted (every query carries `WHERE is_deleted=false`) rather than resurrecting it.

Aligned to what degree? The read path ended up byte-for-byte against upstream. The few intentional differences all sit on the equal-or-more-correct side of the fork (percentiles via `uddsketch` approximation, empty time buckets filled with zeros, stricter 400s on nonsensical dashboard queries), and they're listed in the parity ledger.

## raw_events as SoT: making it actually keep up

This design has a hard cost. Every incoming event means reading that entity's whole history, rebuilding it, and writing it back to the projection and EAV; and since TTL can't truncate history (it has to stay for replay), shedding load by storing less is off the table too. The first backfill load test didn't even finish; the worker stalled partway.

Three bottlenecks turned up. The worst: the ingester's protobuf encoding was synchronous, hogging the worker's event loop and starving the raw_events reads, and only moving it into a worker-thread pool freed it up. The other two were lighter. The raw_events point read degraded into an in-memory scan while data sat unflushed, so I added a timed flush scoped to just that table. And when draining a backlog the same entity got rebuilt over and over, so a watermark skips the redundant rebuilds. After that it drains to completion reliably, but only to "completion" — the write side still needs tuning.

Queries are slower, especially the observation aggregations that go through EAV joins. So the alpha label isn't modesty; write and query both aren't there yet.

What holds up today is two things: it's lean, and it's simple. On my parity/backfill dataset, the same data fit into a single GreptimeDB volume (raw_events + projections + EAV) at roughly an eighth of upstream's footprint (ClickHouse plus a MinIO bucket for event blobs). Don't extrapolate the number, but the direction is clear. And the stack is simpler: one engine holds everything, with no separate object store to operate.

## Performance: two levers, plus a GreptimeDB bug

Normally indexing is the main lever. GreptimeDB's read performance largely hinges on two things landing: Time Index range pruning, and indexed-column pushdown. For a query filtered by time range + an indexed column, if the predicate pushes down into the Region scan, the engine prunes most of the data at the file / row-group level and reads only what it needs. If it doesn't push down, it's a full scan.

The bug we hit sits right on that line. Under load testing, the same predicate prunes fine on a single table, but the moment that table sits under a JOIN, pruning disappears and the scan degrades to a full table scan. It traces to a real GreptimeDB planner bug ([#8338](https://github.com/GreptimeTeam/greptimedb/issues/8338)): when a table filtered by a TIME INDEX (or an indexed column) sits directly under a JOIN, the predicate doesn't push into the Region's `SeqScan`; it lands as a `FilterExec` above the scan, so the scan reads the whole table first and filters after.

`EXPLAIN ANALYZE` makes it obvious. For the same 200 rows:

```sql
-- A) single table: predicate pushes into scan → 1 file_range, 200 rows
SELECT count(*) FROM parent WHERE ts >= '2024-01-30 00:00:00';

-- B) table under JOIN: predicate doesn't push down → 30 file_ranges, 6000 rows (full table)
SELECT count(*) FROM parent p LEFT JOIN child c ON p.k = c.k
WHERE p.ts >= '2024-01-30 00:00:00';

-- C) filter in a subquery first, then JOIN → predicate pushes back down, 1 file_range, 200 rows
SELECT count(*) FROM (SELECT * FROM parent WHERE ts >= '2024-01-30 00:00:00') p
LEFT JOIN child c ON p.k = c.k;
```

A and C both prune to 1 file_range and 200 rows; B scans 30 file_ranges and 6000 rows. At production scale (wide tables of 100K to 1M+ rows), that's the difference between a few-millisecond pruned read and a multi-second full scan. The workaround is to rewrite the SQL: wrap each filtered base table in a subquery so the predicate attaches to the scan before the JOIN (form C). Until the bug is fixed, the app layer just carries this rewrite.

Compaction is the main lever in the backfill case. GreptimeDB's writer flushes roughly once a second under load, so high-frequency writes or a backfill quickly produce a pile of small SST files. And by-type dashboard query latency is dominated by how many SSTs the scan has to merge. Measured on the same query (~3.5M observations): 1022 uncompacted SSTs took 9.6 seconds; after `compact_table` collapsed them into one file, 0.2 seconds. So after a big backfill, compact the hot tables manually:

```sql
ADMIN compact_table('observations_usage_cost', 'strict_window', 86400);
```

## The real reason to fork Langfuse

Up to here, everything Openfuse does is about matching Langfuse. Swapping ClickHouse for GreptimeDB with the same features is, at best, an OpenSearch-to-Elasticsearch story: worth something, not enough.

The real reason is that GreptimeDB is a unified observability database: metrics, logs, traces in one engine, native PromQL / TQL, ingest pipelines, Flow continuous aggregation, native OTLP, object storage underneath. Those capabilities open up a batch of directions that don't grow naturally out of Langfuse's current structure. To be clear up front: none of this ships yet; it's the direction the fork is taking (tracked in [issue #8](https://github.com/tma1-ai/openfuse/issues/8)):

- A PromQL-native metrics layer. Materialize observation cost / usage / latency / error into Prometheus-style metric series; p99 latency, `rate(token_cost[5m])`, aggregation by model / project: PromQL expresses all of that far more naturally than hand-writing two layers of `group by` + `uddsketch`. Langfuse's whole query surface is built on SQL, with no mental model for "pick a series by metric + label, then `rate` / `histogram_quantile`." That's a structural gap, not a flag you flip.
- Expose a metrics endpoint. Let users point their own Grafana / Prometheus at it and treat their LLM app's token / cost / latency / error as ordinary metrics in dashboards they already run. A pure-ClickHouse fork can't offer that.
- PromQL / TQL alerting. Cost spikes, p99 breaches, and rising error rates as threshold rules evaluated in the database, far cheaper than building a product-internal alerting DSL from scratch.
- A native logs view. Bring app / agent logs in alongside traces, correlate logs ↔ traces by `trace_id`, full-text search via `matches_term`. Langfuse has no logs surface.
- Native OTLP ingestion, Flow pre-aggregation, object-storage long retention + downsampling.

Openfuse = running LLM engineering on a real observability database, not one more tool welded to a single store. That's why I'm willing to maintain a fork that diverges permanently from upstream.

## Last, on vibe coding

I ran this migration mostly by directing two agents (Claude Code + Codex), with Copilot reviewing on GitHub, the whole thing coordinated through TMA1. Those two token bills at the top are what TMA1 recorded.

Mostly it was tiring. Not because it was slow, but because it was fast. The agents push quick, so you can't stop; you keep wanting one more step, without the natural breather of waiting on a compile or a test.

Quality beat my expectations. Claude Code / Codex writing, Copilot reviewing, TMA1 holding context across sessions: the combination delivered well. With a few conditions: smoke-test early, verify the critical paths by hand, diff byte-for-byte against upstream. The gap between "looks right" and "is right" that an agent hands you, you have to cross yourself. The read path matching upstream byte-for-byte didn't come from trusting the agents; it came from smoke tests and a parity harness.

Whether AI can pull off a big rewrite comes down not to how strong the model is but to **engineering discipline**: break milestones down, realign plan and implementation regularly, put a check gate at every critical point. Do that and delivery holds up; skip it and you've just handed a one-liner to an agent and taken on the debt.

Openfuse is still alpha. Code and issues are on [GitHub](https://github.com/tma1-ai/openfuse). Come kick the tires.
