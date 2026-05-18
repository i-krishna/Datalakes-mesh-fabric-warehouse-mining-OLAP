# Lakehouse architecture 
<img width="562" height="450" alt="image" src="https://github.com/user-attachments/assets/b8c9586d-9f29-4e30-842a-94fad0748ad7" />

#  From Application to Lakehouse: Code-First Microservices as the Data Ingestion Layer

<img width="471" height="324" alt="image" src="https://github.com/user-attachments/assets/c0cf77a7-71ea-4eed-b0d0-dc4ef4b1f0e2" />

Most data architecture diagrams start at the lakehouse. But data starts in applications. This section bridges the two — showing how a code-first microservices backend (ASP.NET Core + EF Core) produces structured, version-controlled data that flows cleanly into a lakehouse architecture.

Each microservice owns its own database schema, defined in C# and evolved via EF Core migrations — not hand-written DDL, not shared schema files. This is the application-side equivalent of Iceberg's schema evolution: where domain model changes are tracked, reversible, and tested before they ever touch production.

The pattern has three layers working together: see [aspnetcore-efcore-db-webapi](https://github.com/i-krishna/aspnetcore-efcore-db-webapi 

**Layer 1** — Code-first schema management (application side). When a software engineer adds a field in AddCustomerField migration, that change is visible in Git history — not discovered at 2 AM when the Spark ETL breaks. The EF Core __EFMigrationsHistory table becomes an audit trail that data engineers can use to version-gate their pipelines. 

The C# domain model is the source of truth for schema. Product.cs defines a class; EF Core generates a Migration file (Up() / Down()) that maps to DDL. The __EFMigrationsHistory table tracks applied migrations. This makes schema changes reviewable in Git, testable in CI, and reversible — the same guarantees you get from Iceberg's snapshot-based schema evolution, but at the application layer.

**Layer 2** — Loose coupling via service boundaries. Each microservice owns exactly one database. No service reads another's SQL tables directly. Communication happens via RESTful APIs (synchronous queries) or domain events / CDC (async data propagation). Engineering teams can deploy independently, roll back independently, and evolve their schemas without co-ordinating with the data platform team.

Loose coupling between app and platform: The RESTful API layer acts as the contract boundary. The data platform consumes events or snapshots from these APIs — it never directly couples to the application's SQL Server schema. This mirrors the same principle as the Iceberg REST Catalog in the next section below: engines talk to a catalog endpoint, not directly to storage.

**Layer 3** — Event-driven ingestion into the lakehouse. The microservices' transactional databases (SQL Server) become the bronze source. Change Data Capture (CDC) — via Debezium, Azure Event Hubs, or similar — captures row-level mutations and streams them to the lakehouse bronze layer. From there, Spark/Databricks pipelines clean into silver and aggregate into gold. The __EFMigrationsHistory table provides the data engineering team a reliable audit trail: when a new column appears in a migration commit, they know exactly when to update their bronze-to-silver transformation pipeline

CI/CD readiness: EF Core migrations are designed to run in a CI/CD pipeline (dotnet ef database update). This means schema changes go through the same pull request → review → deploy lifecycle as application code (Azure DevOps) — giving data teams predictable, tested schema evolution rather than surprise ALTER TABLE statements in production.

# How Iceberg REST Catalog Replaces Fragmented Metastores (One Table, Three Engines)

<img width="405" height="448" alt="image" src="https://github.com/user-attachments/assets/46d37cf0-d547-4004-9916-979509bb82d0" />

Example: A retailer like Target has three teams all working with the same silver.payments table:

The data engineering team uses Spark on Databricks to run nightly ETL jobs
The finance team uses Snowflake to run ad-hoc revenue queries
The data science team uses Trino to build ML features

## In the Hive Metastore world:
Spark owns a Hive Metastore. The finance team copies data nightly into Snowflake's internal storage. The data science team runs a separate Hive Metastore for Trino. Three copies of the same data, three schemas that drift apart, three access control lists that contradict each other. When the data engineering team adds a new column at 2 AM, Snowflake doesn't know until someone manually re-exports the data — which takes hours.

## In the Iceberg REST Catalog world:
All three teams point their engines at one Polaris catalog (or Unity Catalog or Glue Catalog) over HTTPS. Polaris says silver.payments lives at s3://data/silver/payments/metadata/v00042.json. When Spark commits a new snapshot at 2 AM, the metadata pointer atomically updates to v00043.json. The whole operation takes 200 milliseconds. At 2:01 AM, when the finance analyst runs a Snowflake query, it calls the same Polaris REST endpoint, gets v00043.json, and reads the same Parquet files Spark just wrote. Trino runs a feature query. Same Polaris endpoint. No copy. No export. No delay. No schema drift. One table. One truth. Three different engines reading it simultaneously - all in milliseconds.

<img width="1440" height="950" alt="image" src="https://github.com/user-attachments/assets/a2327834-0e98-45e1-a577-accc90fe232c" />

# Partitioning vs Delta lake's Clustering vs Iceberg's v3 clustering

<img width="636" height="481" alt="image" src="https://github.com/user-attachments/assets/194dea48-3e44-4ad1-a59d-9d9c80dd1b9d" />

```
-- Old world: one rigid choice made at creation
CREATE TABLE orders
PARTITIONED BY (date)   -- locked forever
STORED AS PARQUET;

-- Consequences:
-- Query on date → great (partition pruning works)
-- Query on region → terrible (full scan, all partitions)
-- Query on customer_id → terrible (same problem)
-- Want to change partition column → full table rewrite
```
```
-- Delat lake: Z-order maps multi-dimensional data to 1D
so similar values on ALL cluster columns
end up in the same files.

region=APAC, category=Electronics → Z-value: 0010110
region=APAC, category=Books       → Z-value: 0010101
region=EMEA, category=Electronics → Z-value: 1010110

Files are sorted by Z-value, so APAC rows
cluster together regardless of date.
```

```
Old Delta approach:
OPTIMIZE table ZORDER BY (region, category)
→ one-time rewrite, no ongoing awareness
→ new data written without Z-ordering
→ gradually degrades as unoptimized files accumulate
→ must manually re-run OPTIMIZE regularly

Liquid clustering:
ALTER TABLE ... CLUSTER BY (region, category)
→ table "knows" its clustering columns permanently
→ new writes try to respect clustering automatically
→ background OPTIMIZE incrementally maintains it
→ can change clustering columns without full rewrite
→ "liquid" because it adapts as data and queries evolve
```

ALTER TABLE gold.shipments
CLUSTER BY (shipping_region, carrier_tier, shipment_date);

OPTIMIZE gold.shipments;

# Data compute vs AI Compute

See [here](https://github.com/i-krishna/Business-Analytics/blob/main/Data-Science/Python/matrix_multiply_inverse.py#L2) for when to use Data compute vs AI Compute 

For data compute (Spark), the bottleneck isn't RAM — it's the network and disk:

Spark bottleneck chain:
S3 (object storage)
    ↓  ~10 GB/s (network)        ← usually the real bottleneck
Node RAM (DDR5)
    ↓  ~300 GB/s
CPU L3 cache
    ↓  ~1,000 GB/s
CPU cores

Spark jobs spend most of their time waiting for data to arrive from S3 over the network. The CPU cores are fast enough — they're just starved of data. This is why Spark parallelism (adding more machines) helps so much — you're adding more network connections to S3, not faster CPUs.

## CPU tenant isolation 
```
┌─────────────────────────────────────────────────┐
│ LEVEL 4: Kubernetes (orchestration)             │
│ ResourceQuota, LimitRange, Namespaces           │
│ "Customer 1 gets max 16 cores cluster-wide"     │
├─────────────────────────────────────────────────┤
│ LEVEL 3: cgroups v2 (Linux kernel)              │
│ Enforces CPU throttle + memory hard limits      │
│ "This container cannot use > 4 cores right now" │
├─────────────────────────────────────────────────┤
│ LEVEL 2: Linux namespaces (kernel)              │
│ Process, network, filesystem visibility         │
│ "This container cannot SEE other containers"    │
├─────────────────────────────────────────────────┤
│ LEVEL 1: CFS scheduler (Linux kernel)           │
│ Decides which process runs on which core        │
│ "Fair share of CPU time across all processes"   │
├─────────────────────────────────────────────────┤
│ LEVEL 0: Physical hardware (shared)             │
│ Cores, RAM, L3 cache — NO hardware partitioning │
│ "Everything shares the same silicon"            │
└─────────────────────────────────────────────────┘
```
The critical insight: every layer above Level 0 is software. The hardware itself is fully shared. This is the fundamental difference from GPU MIG.

```
GPU MIG              CPU (cgroups + K8s)
─────────────────────────────────────────────────────────
Partition unit    Hardware slice        Software container
Memory isolation  Physical (MMU)        Logical (cgroup limit)
CPU isolation     Dedicated SMs         Time-shared (throttle)
Cache isolation   L2 partitioned        L3 fully shared ← weak
Network isolation Dedicated NVLink      Shared NIC (QoS optional)
Noisy neighbour   Impossible            Possible (cache, I/O)
Strongest form    7x MIG on one A100    Dedicated node per tenant
Cost efficiency   7 tenants, 1 GPU      Many tenants, shared nodes
Use case          Inference serving     ETL, batch, data pipelines
```
