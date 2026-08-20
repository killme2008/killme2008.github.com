+++
title = "Twenty Years of S3: How It Became the Default Persistence Layer for Modern Data Infrastructure"
date = 2026-08-20
description = "S3 launched twenty years ago as a simple object store with eventual consistency. Strong consistency, conditional writes, and Express One Zone turned it into the durable source of truth for warehouses, streaming systems, databases, and now Git hosting."

[taxonomies]
tags = ["Object Storage", "Architecture", "Distributed Systems", "Cloud"]

[extra]
toc = true
social_media_card = "/images/s3-at-twenty-cover.jpg"
+++

![Twenty years of S3](/images/s3-at-twenty-timeline.webp)

Cursor published [a piece](https://cursor.com/blog/git-at-any-scale) this week on Continuity, the Git storage system behind its Origin hosting platform. The design would have sounded strange not long ago.

Continuity keeps a write-ahead log on S3 and does not acknowledge a push until the data is durable there. The Git repositories on local NVMe are warm caches. Any server can accept a push; rendezvous hashing picks the preferred primary, but correctness does not depend on that choice. Concurrent updates are serialized through S3's atomic conditional writes.

Cursor reports 120 pushes per second on S3 Standard, more than 300 on S3 Express One Zone, and read throughput that continues to scale through 100 replicas.

This is Git hosting: small objects, frequent writes, and a hard requirement for linearizable updates. It is not the large-object, sequential-access workload that people usually reach for when explaining why S3 works well.

Cursor's post is a good excuse to ask how we got here. S3 launched twenty years ago as a simple object store with eventual consistency. It is now the durable source of truth for data warehouses, lakehouses, streaming systems, databases, and, apparently, Git hosting.

That change did not happen all at once. The economics came first. The semantics and latency took much longer.

## What "the persistence layer" means

Calling S3 the persistence layer does not mean sending every read and write on the hot path directly to S3.

Continuity is a clean example. S3 owns durability and the authoritative order of updates. Local NVMe makes Git operations fast. If a local copy disappears, the system can rebuild it from the WAL. The cache is important for performance, but it is not part of the durability model.

This separation now appears across modern data infrastructure. Object storage holds the authoritative data. Compute nodes keep local caches, indexes, or materialized state that can be discarded and rebuilt. Once the local disk stops being the source of truth, compute can scale, fail, and move independently of stored data.

![S3 as the persistence layer](/images/s3-at-twenty-persistence-layer.webp)

That is the architectural shift this article is about.

## The economics forced the issue

Cloud compute became elastic. Block storage remained something you provision, attach, and pay for while it exists. That mismatch matters for multi-tenant systems with bursty traffic and large amounts of idle data.

The raw storage prices in us-east-1 make the starting point clear:

| Tier | Service | Price per GB-month |
|------|---------|--------------------|
| Object storage, hot | S3 Standard | $0.023 |
| Object storage, cold | S3 Glacier Deep Archive | $0.00099 |
| Block storage, throughput HDD | EBS st1 | $0.045 |
| Block storage, general SSD | EBS gp3 | $0.08 |
| Block storage, performance SSD | EBS io2 | $0.125 |

These are capacity prices, not a total-cost comparison. S3 charges for requests, retrieval, and data transfer. A system built on it also needs caches, compaction, and metadata management. At high write rates, small-object request charges can exceed the storage bill.

Even with those costs, the underlying difference remains: EBS bills provisioned capacity, while S3 bills stored bytes. gp3 costs about 3.5 times as much per GB as S3 Standard, and io2 costs more than five times as much before additional IOPS and throughput charges. Provisioned block devices make sense for active state. They are an expensive place to keep cold capacity or reserve for peaks that may never arrive.

Once Kubernetes made disposable compute normal, data systems needed a storage layer that could outlive any node, serve many tenants, and grow without capacity planning. S3 already had that cost model. It still needed better semantics.

## How S3 filled in the missing pieces

### 2006–2019: scale and cost, without database semantics

S3 launched in 2006 with GET, PUT, DELETE, buckets, and eventual consistency. It deliberately did not provide POSIX semantics. That trade made enormous scale possible, but it also limited what systems could safely build on top.

The next decade filled in the operational basics. Versioning arrived in 2010, followed by multipart upload and server-side encryption. VPC endpoints and Standard-IA appeared in 2015. Automatic prefix scaling in 2018 removed the old requirement to randomize key prefixes, while Intelligent-Tiering made storage-class management less manual.

Analytical systems did not wait for perfect semantics. Snowflake separated compute from S3-backed storage in its 2016 architecture. Iceberg, Delta Lake, and later Hudi made object storage the durable home of table data while adding transaction metadata above it.

But object storage still had a semantic hole. A successful write did not guarantee that the next reader, or even the next LIST, would immediately observe it. Systems compensated with DynamoDB indexes, S3Guard, EMRFS Consistent View, or their own metadata services. S3 held the bytes, but it could not yet carry all the coordination.

### 2020: strong consistency

In December 2020, S3 added strong read-after-write consistency for GET, PUT, and LIST. After a successful write, readers could immediately observe the latest object and listings reflected the change.

EMRFS Consistent View, S3Guard, and their DynamoDB indexes became dead weight. Systems built from immutable files and mutable manifests could finally treat the object store as a current, authoritative view rather than one that would catch up eventually.

Strong consistency was necessary, but not sufficient. S3 was still slow compared with a local SSD, and applications could not atomically reject an update based on the version they had read.

### 2023: latency stopped being an automatic disqualifier

S3 Express One Zone launched in November 2023 with consistent single-digit-millisecond access inside one Availability Zone. It trades multi-AZ resilience for latency, so it is not a drop-in replacement for S3 Standard. For request-heavy systems that already manage redundancy at another layer, the trade can be useful.

Mountpoint for S3 also became generally available that year. It provided a file API for read-heavy applications without pretending to be a full POSIX filesystem: no in-place modification, no POSIX locking, and only limited rename and append support in directory buckets.

Neither product turned S3 into an NVMe drive. They did make the latency and interface objections workload-specific rather than universal.

### 2024: S3 learned compare-and-swap

Conditional writes completed the more important part of the story.

S3 added create-if-absent writes with `If-None-Match` in August 2024, then ETag-based compare-and-swap with `If-Match` in November. Bucket policies can require these preconditions, preventing a client from silently bypassing them.

With strong consistency and conditional writes on the same key, a system can implement operations such as manifest commits, WAL index updates, and catalog pointer changes directly against S3. The coordination has not disappeared; S3 now performs the atomic serialization that previously required DynamoDB, Postgres, or a consensus service beside the object store.

This is the capability Continuity uses. A push first becomes durable as a WAL object. Publishing it requires a conditional update to the WAL index. If two servers race, one wins and the other retries against the new version. There is no repository-level leader election on the correctness path.

That design was not possible on S3 before conditional writes.

### 2025–2026: S3 expanded beyond byte storage

Consistency and conditional writes are what make the Continuity design possible. The additions that followed are different: they show how much more of the data layer AWS wants S3 to own.

S3 Vectors became generally available in December 2025. It added native vector indexes and similarity queries, with up to two billion vectors per index and twenty trillion per bucket. Query latency starts around 100 milliseconds for frequently accessed indexes and remains under a second for infrequent queries. That is not a replacement for every vector database, but it is a new workload running inside S3 rather than merely storing files underneath one.

S3 Files followed in April 2026. Built with Amazon EFS, it exposes S3 data over NFS 4.1 and 4.2 with file-system semantics and low latency for active files. Writes reach the file system immediately but are exported back to the bucket after a period of inactivity, so applications that mix NFS and S3 API access need to account for that synchronization window.

In June 2026, S3 added Annotations: up to 1,000 mutable metadata payloads per object, each independently addressable and queryable through S3 Metadata tables. Tags were useful for lifecycle and access policies; annotations can carry much larger application context without rewriting the object.

Vectors, Files, and Annotations solve different problems, and none of them makes S3 a database by itself. But put them next to strong consistency, conditional writes, and Express One Zone, and the original contract is almost hard to recognize. The old description of S3 as a slow bucket for immutable blobs no longer fits.

## Systems stopped waiting

The warehouse and lakehouse world went first. Snowflake, Databricks, ClickHouse Cloud, and the Iceberg/Delta/Hudi ecosystem all treat object storage as durable data and keep compute replaceable. Streaming systems followed: AutoMQ and WarpStream rebuilt Kafka-compatible services around object storage instead of fleets of brokers holding authoritative local disks.

Neon took the same separation into Postgres. Compute nodes are disposable. Safekeepers protect recent WAL, while Pageservers turn it into immutable layers, upload those layers to object storage, and use local SSD to serve the active working set. The Neon-based architecture now powers Databricks Lakebase.

Continuity matters because it removes the usual escape clause. Git hosting is not an analytical scan over large Parquet files. A push updates small pieces of shared state and must be immediately visible. Continuity persists each push to S3, serializes publication with conditional writes, and validates replicas against the authoritative WAL index before serving reads. Cursor reports that a conditional GET takes less than 10 milliseconds on average.

The local repositories still matter. Git wants random access to packfiles, and NVMe remains the right place to execute those operations. What changed is the recovery and consistency boundary. Local repositories can be created, replicated, compacted, or deleted without becoming the source of truth.

That pattern scales down as well as up. A busy monorepo can have many read replicas. An idle repository can have none until the next request rematerializes it. The durable copy does not force a minimum fleet of continuously provisioned replicas.

![Modern data infrastructure built around S3](/images/s3-at-twenty-data-infrastructure.webp)

## The rest of the storage engine still matters

Putting authoritative data on S3 does not make the rest of the storage engine disappear.

Latency-sensitive systems still need memory and local SSD for their active working sets. Small writes usually need batching, and immutable objects eventually need compaction. LIST is not a substitute for a low-latency catalog or secondary index. Request charges can dominate workloads with tiny objects, and cross-cloud egress can erase the storage savings. S3 Express One Zone also has a different failure boundary from S3 Standard: the lower latency comes with single-AZ placement.

S3 Files adds another consistency boundary because file-system writes are not exported synchronously to the bucket. S3 Vectors targets cost-sensitive, subsecond retrieval rather than the lowest possible query latency. These are design constraints, not footnotes.

The architecture works when the system is honest about them. S3 provides durable, elastic shared state. Caches, compaction, indexes, and catalogs turn that state into a usable database.

## The new default

The interesting change over the past twenty years is not that S3 became fast enough to replace every disk. It did not.

S3 became good enough to own durability and authoritative state for systems that used to require provisioned block storage or a replicated local filesystem. Strong consistency removed stale reads. Conditional writes made S3 usable for linearizable metadata updates. Express One Zone brought request latency into range for workloads that could not tolerate S3 Standard. The cloud cost model did the rest.

By 2026, if a new cloud data infrastructure system does not use S3 as its source of truth, that should be a workload-specific decision rather than an inherited default.

GreptimeDB has followed this architecture since day one. Time-series and observability data were an early fit because writes are append-heavy, volume is difficult to predict, and retention spans hot and cold data. GreptimeDB persists immutable table data to object storage, uses local disk as a buffer and cache, and keeps unflushed state in its WAL. Storage capacity is not tied to a fixed compute fleet.

Cursor's Continuity shows that the pattern has moved well beyond analytical data. A workload once considered too small-write-heavy, latency-sensitive, and consistency-sensitive for S3 is now built around it.

That is a more useful measure of S3's first twenty years than the number of features it has accumulated.

## References

- [Git at any scale](https://cursor.com/blog/git-at-any-scale) (Cursor, 2026-08-18)
- [Snowflake's Elastic Data Warehouse](https://dl.acm.org/doi/10.1145/2882903.2903741) (SIGMOD 2016)
- [Lakehouse: A New Generation of Open Platforms](https://www.cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf) (CIDR 2021)
- [VPC Endpoint for Amazon S3](https://aws.amazon.com/blogs/aws/new-vpc-endpoint-for-amazon-s3/) (2015-05)
- [S3 increased request rate performance](https://aws.amazon.com/about-aws/whats-new/2018/07/amazon-s3-announces-increased-request-rate-performance/) (2018-07)
- [S3 strong read-after-write consistency](https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/) (2020-12)
- [S3 Express One Zone](https://aws.amazon.com/s3/storage-classes/express-one-zone/)
- [Mountpoint for S3 GA](https://aws.amazon.com/blogs/aws/mountpoint-for-amazon-s3-generally-available-and-ready-for-production-workloads/) (2023-08)
- [Mountpoint for S3 file system semantics](https://github.com/awslabs/mountpoint-s3/blob/main/doc/SEMANTICS.md)
- [S3 conditional writes, `If-None-Match`](https://aws.amazon.com/about-aws/whats-new/2024/08/amazon-s3-conditional-writes/) (2024-08)
- [S3 conditional writes, `If-Match`](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-s3-functionality-conditional-writes/) (2024-11)
- [Amazon S3 Vectors GA](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-vectors-generally-available/) (2025-12)
- [Amazon S3 Files GA](https://aws.amazon.com/about-aws/whats-new/2026/04/amazon-s3-files/) (2026-04)
- [S3 Files synchronization](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-synchronization.html)
- [Amazon S3 Annotations](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-s3-annotations-business-context/) (2026-06)
- [S3 storage classes](https://aws.amazon.com/s3/storage-classes/)
- [Neon architecture](https://neon.com/docs/introduction/architecture-overview)
- [Lakebase's Neon-based architecture](https://developers.databricks.com/perspectives/what-database-architecture-separates-compute-from-storage-for-postgresql)
- [ClickHouse Cloud stateless compute architecture](https://clickhouse.com/blog/clickhouse-cloud-stateless-compute)
- [AutoMQ S3 storage architecture](https://docs.automq.com/automq/architecture/s3stream-shared-streaming-storage/s3-storage)
- [WarpStream: Kafka is dead, long live Kafka](https://www.warpstream.com/blog/kafka-is-dead-long-live-kafka)
- [GreptimeDB architecture](https://docs.greptime.com/user-guide/concepts/architecture)
