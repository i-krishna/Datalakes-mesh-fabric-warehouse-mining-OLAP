# Lakehouse architecture 
<img width="562" height="450" alt="image" src="https://github.com/user-attachments/assets/b8c9586d-9f29-4e30-842a-94fad0748ad7" />

#  From Application to Lakehouse: Code-First Microservices as the Data Ingestion Layer

<img width="471" height="324" alt="image" src="https://github.com/user-attachments/assets/c0cf77a7-71ea-4eed-b0d0-dc4ef4b1f0e2" />

Most data architecture diagrams start at the lakehouse. But data starts in applications. This section bridges the two — showing how a code-first microservices backend (ASP.NET Core + EF Core) produces structured, version-controlled data that flows cleanly into a lakehouse architecture.

Each microservice owns its own database schema, defined in C# and evolved via EF Core migrations — not hand-written DDL, not shared schema files. This is the application-side equivalent of Iceberg's schema evolution: where domain model changes are tracked, reversible, and tested before they ever touch production.

The pattern has three layers working together: see project implementation [aspnetcore-efcore-db-webapi](https://github.com/i-krishna/aspnetcore-efcore-db-webapi )

**Layer 1** — Code-first schema management (application side). When a software engineer adds a field in AddCustomerField migration, that change is visible in Git history — not discovered at 2 AM when the Spark ETL breaks. The EF Core __EFMigrationsHistory table becomes an audit trail that data engineers can use to version-gate their pipelines. 

The C# domain model is the source of truth for schema. Product.cs defines a class; EF Core generates a Migration file (Up() / Down()) that maps to DDL. The __EFMigrationsHistory table tracks applied migrations. This makes schema changes reviewable in Git, testable in CI, and reversible — the same guarantees you get from Iceberg's snapshot-based schema evolution, but at the application layer.

**Layer 2** — Loose coupling via service boundaries. Each microservice owns exactly one database. No service reads another's SQL tables directly. Communication happens via RESTful APIs (synchronous queries) or domain events / CDC (async data propagation). Engineering teams can deploy independently, roll back independently, and evolve their schemas without co-ordinating with the data platform team.

Loose coupling between app and platform: The RESTful API layer acts as the contract boundary. The data platform consumes events or snapshots from these APIs — it never directly couples to the application's SQL Server schema. This mirrors the same principle as the Iceberg REST Catalog in the next section below: engines talk to a catalog endpoint, not directly to storage.

**Layer 3** — Event-driven ingestion into the lakehouse. The microservices' transactional databases (SQL Server) become the bronze source. Change Data Capture (CDC) — via Debezium, Azure Event Hubs, or similar — captures row-level mutations and streams them to the lakehouse bronze layer. From there, Spark/Databricks pipelines clean into silver and aggregate into gold. The __EFMigrationsHistory table provides the data engineering team a reliable audit trail: when a new column appears in a migration commit, they know exactly when to update their bronze-to-silver transformation pipeline

CI/CD readiness: EF Core migrations are designed to run in a CI/CD pipeline (dotnet ef database update). This means schema changes go through the same pull request → review → deploy lifecycle as application code (Azure DevOps) — giving data teams predictable, tested schema evolution rather than surprise ALTER TABLE statements in production.

<img width="632" height="346" alt="image" src="https://github.com/user-attachments/assets/43e47374-026b-4dce-85c6-f44759c54d58" />

# EF Migrations vs Data Contracts 

EF migrations (code-first vs database-first) decide how your schema gets built.
Data contracts decide who owns it, who depends on it, and how changes get communicated.

## Project Story - Two teams in a banking project use one database with zero communication of changes 
LoanOriginationTeam owns the StudentLoans table. They use Entity Framework with a code-first approach — developers write C# model classes, and EF generates the SQL migrations automatically. It's clean, it's developer-friendly, and it works great for their team.

AnalyticsTeam reads from that same StudentLoans table to power the CFO's weekly dashboard. They don't own the table. They just depend on it.

There is no formal agreement between the two teams about what the schema looks like, who can change it, or what happens when it changes.

**Act 1** — The Migration That Broke Everything
On Tuesday morning, A developer on LoanOriginationTeam gets a ticket: "Regulatory requirement — the CFPB now requires us to store APR separately from interest rate."
They open their StudentLoan.cs model and add a field:
```
csharppublic class StudentLoan
{
    public int Id { get; set; }
    public string StudentId { get; set; }
    public decimal PrincipalAmount { get; set; }
    public decimal InterestRate { get; set; }           // old field — still here
    public decimal AnnualPercentageRate { get; set; }   // new field — APR per CFPB
}
```
They run the EF migration command:
```
bash
dotnet ef migrations add AddAnnualPercentageRate
dotnet ef database update
```
EF generates and applies this SQL:
```
sql
ALTER TABLE StudentLoans
ADD AnnualPercentageRate DECIMAL(5,3) NOT NULL DEFAULT 0;
```
Code compiles. Migration runs clean. Tests pass. The developer ships it to production Wednesday morning, proud of a clean regulatory fix.

Meanwhile, the AnalyticsTeam's dashboard is still querying InterestRate. The loan origination system starts writing the real rate into AnnualPercentageRate and stops updating InterestRate. The two columns silently diverge.

Nobody notices until Friday, when the CFO opens the weekly dashboard and the average interest rate column shows 0.000% across 40,000 loans.

The CFO sends one Slack message: "Is our loan portfolio actually at zero percent interest?"

**Act 2 **— Code-First vs Database-First Isn't the Problem

The post-mortem meeting starts with the wrong question: "Should we have used database-first instead of code-first?" Neither. 

Code-First means developers write the C# model, and EF generates the database schema from it. The C# class is the source of truth. Schema changes flow from code to database.

```
c#
// Developer adds a property here...
public decimal AnnualPercentageRate { get; set; }

// ...and EF generates the migration automatically
// ALTER TABLE StudentLoans ADD AnnualPercentageRate DECIMAL(5,3)
```

Database-First means a DBA writes the SQL schema directly, and EF reverse-engineers C# model classes from it. The database is the source of truth. Code follows the database.
```
bash

# DBA writes SQL, then developer runs:
dotnet ef dbcontext scaffold "connection_string" \
  Microsoft.EntityFrameworkCore.SqlServer \
  --output-dir Models
# EF generates the C# class from whatever the DB looks like
```

Both approaches are valid. Both have legitimate use cases. Code-first gives developers full control and clean versioned migrations. Database-first is better when the database already exists or when DBAs need to own the schema for performance or compliance reasons.

But here's the thing — neither approach stopped the incident. The problem wasn't how the schema was created. The problem was that nobody told the AnalyticsTeam it was changing.

**Act 3** — The Real Problem is a Missing Contract
What the two teams needed was a data contract — a formal, versioned agreement about what the StudentLoans table contains, who can change it, and what happens when it changes.

A data contract is not code. It's not a migration. It's a document that lives in Git alongside the code, that both teams have agreed to, and that neither team can silently violate.

Here's what the contract for StudentLoans should have looked like:

```
yaml
# contracts/student_loans.yaml
name: StudentLoans
version: "2.1"
owner: LoanOriginationTeam
effective_date: 2024-01-01

schema:
  - column: Id
    type: int
    nullable: false
    description: "Primary key"

  - column: StudentId
    type: string
    nullable: false
    pii: true
    description: "Unique student identifier — PII protected"

  - column: PrincipalAmount
    type: decimal(10,2)
    nullable: false
    bounds: [1000, 500000]
    description: "Loan principal in USD"

  - column: InterestRate
    type: decimal(5,3)
    nullable: false
    bounds: [0.0, 12.5]
    description: "Annual interest rate as a percentage"

consumers:
  - name: AnalyticsTeam
    access: SELECT
    columns: [StudentId, PrincipalAmount, InterestRate]
    row_filter: "status = 'FUNDED'"

  - name: ComplianceReporting
    access: SELECT
    columns: [all]
    row_filter: none

quality_rules:
  - "PrincipalAmount > 0"
  - "InterestRate BETWEEN 0.0 AND 12.5"
  - "DisbursementDate <= CURRENT_DATE"

freshness: "15 minutes"
```

With this contract in place, the process for adding AnnualPercentageRate would look completely different.

**Act 4** — The Same Change, Done Right
The developer on LoanOriginationTeam gets the same CFPB ticket. This time, before touching a single line of C# code, they open contracts/student_loans.yaml and write a proposed change:

```
yaml
# contracts/student_loans.yaml — proposed version 2.2
version: "2.2"
changes:
  - type: column_added
    column: AnnualPercentageRate
    type: decimal(5,3)
    nullable: false
    bounds: [0.0, 25.0]
    description: "CFPB-compliant APR — replaces InterestRate for regulatory reporting"
    replaces: InterestRate
    reason: "CFPB regulatory requirement effective 2024-06-01"
    consumers_affected:
      - AnalyticsTeam
      - ComplianceReporting
    migration_plan: >
      InterestRate will remain populated for 60 days after deployment.
      AnnualPercentageRate will be computed from InterestRate at migration time.
      After 60 days, InterestRate will be deprecated.
```
They open a pull request — not just for the C# model change, but for the contract change too. The CI/CD pipeline has a contract validation step:

```
yaml
# .github/workflows/validate-contract.yml
- name: Validate data contract
  run: |
    python scripts/validate_contract.py \
      --contract contracts/student_loans.yaml \
      --migration migrations/AddAnnualPercentageRate.sql \
      --require-consumer-approval AnalyticsTeam ComplianceReporting
```
The pipeline blocks the merge until both AnalyticsTeam and ComplianceReporting have reviewed and approved the contract change. Not just the code — the contract.

AnalyticsTeam sees the PR. They update their dashboard query from InterestRate to AnnualPercentageRate before the migration even runs. They add a comment: "Dashboard updated in commit abc123. Approved."
ComplianceReporting does the same.

LoanOriginationTeam merges the PR. The EF migration runs in production. Both consuming teams are already updated and ready. The CFO's Friday dashboard shows correct numbers.

No incident. No post-mortem. 

**Act 5** — This is Data DevOps
What the second workflow describes is data DevOps — applying software engineering discipline to data changes. Just like a code review ensures a developer's change doesn't break another developer's feature, a data contract review ensures a schema change doesn't break another team's pipeline.
The full data DevOps pipeline looks like this:

```
Developer updates C# model (code-first)
           │
           ▼
EF generates migration SQL
           │
           ▼
Developer updates data contract YAML
           │
           ▼
Pull request opened
           │
           ├── CI: Does migration match contract?
           ├── CI: Are all affected consumers listed?
           ├── Review: AnalyticsTeam approves
           └── Review: ComplianceReporting approves
                       │
                       ▼ (all gates pass)
           Migration deployed to production
           Contract version bumped to 2.2
           CHANGELOG updated automatically
```

The contract is version-controlled. It has a changelog. Consumers can pin to a contract version just like they pin to a package version. If LoanOriginationTeam needs to make a breaking change, they bump the major version (2.x → 3.0) and give consumers a migration window — just like a public API would.

<img width="476" height="312" alt="image" src="https://github.com/user-attachments/assets/00720e5e-6af6-49b8-bbe6-216b4385e8d0" />

Code-first vs database-first is a team workflow decision. It answers: "Where does the schema definition live — in C# or in SQL?"

Data contracts is a governance decision. It answers: "Who gets a say when the schema changes, and how do we make sure nobody's pipeline breaks silently?"

You can use code-first and data contracts. You can use database-first and data contracts. The contract is the communication layer that sits on top of whichever EF strategy you choose. It's the team agreement that makes incidents like above one impossible — not by preventing schema changes, but by making sure every team that depends on the schema sees the change coming, agrees to it, and is ready for it before it ships.

The CFO's dashboard showed correct numbers. The post-mortem never happened. That's the goal.

# Data Contracts vs Catalogs 

Continuing from the EF Migrations story.
We ended with a YAML file in Git called contracts/StudentLoans.yaml.
That file is just a promise written down. This story is about what makes it real.

An above COntract.YAML file in Git defines what StudentLoans contains, who owns it, and what must happen before anyone changes it. However, team need something to enforce that contract — because a YAML file in Git doesn't block bad queries or mask SSNs by itself. That second thing is a data catalog. Specifically, Unity Catalog (Databricks) or Polaris (Snowflake's open catalog).

The Contract is the Promise. The Catalog is the Implementation of Promise. For example, 
- Data contract (YAML in Git) = the law written in the books.
"AnalyticsTeam may only read StudentId, PrincipalAmount, and InterestRate. SSNs must be masked. Data must be no older than 15 minutes."
- Unity Catalog / Polaris = the sheriff enforcing that law.
When AnalyticsTeam runs a query, the catalog checks: Is this role allowed? Which columns? Is the data stale? Mask the SSN. Block the rest.

This Contract.yaml file lives in Git. It gets reviewed in pull requests. Both producer and consumer teams sign off on it — exactly like the EF migration story. But by itself it doesn't stop anyone from doing anything.

- What Unity Catalog Does With It
Once the contract is agreed on, someone registers the table in Unity Catalog and wires up the rules:
```
sql-- Register the table
CREATE TABLE main.finance.StudentLoans (
  StudentId      STRING,
  PrincipalAmount DECIMAL(10,2),
  InterestRate   DECIMAL(5,3)
);

-- Mark StudentId as PII (per the contract)
ALTER TABLE main.finance.StudentLoans
  MODIFY COLUMN StudentId
  SET TAG governance.pii = 'true';

-- Mask StudentId for anyone not on the allow list
CREATE MASKING POLICY governance.pii_mask
  AS (val STRING) RETURNS STRING ->
    CASE
      WHEN CURRENT_ROLE() IN ('COMPLIANCE_TEAM', 'LOAN_ORIGINATION_TEAM')
        THEN val
      ELSE '***-***-****'
    END;

ALTER TABLE main.finance.StudentLoans
  MODIFY COLUMN StudentId
  SET MASKING POLICY governance.pii_mask;

-- Set freshness SLA as table property
ALTER TABLE main.finance.StudentLoans
  SET TBLPROPERTIES ('freshness_sla' = '15 minutes');
```

Now when AnalyticsTeam queries StudentLoans, Unity Catalog automatically:

Returns ***-***-**** for StudentId — they never see the real value
Checks the last refresh timestamp — if it's more than 15 minutes old, the catalog flags it as stale
Logs every query to SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY for the audit trail

The YAML contract became operational policy inside the catalog.

**What Polaris Does Differently**
Polaris (open-sourced by Snowflake, now Apache) does the same job but for Iceberg tables, via REST:
```
json
POST /v1/namespaces/finance/tables
{
  "name": "StudentLoans",
  "schema": {
    "fields": [
      {"name": "StudentId",       "type": "string",      "required": true},
      {"name": "PrincipalAmount", "type": "decimal(10,2)","required": true},
      {"name": "InterestRate",    "type": "decimal(5,3)", "required": true}
    ]
  },
  "properties": {
    "owner":          "LoanOriginationTeam",
    "freshness_sla":  "15 minutes",
    "pii_columns":    ["StudentId"]
  }
}
```
Polaris's headline feature is credential vending — instead of handing engines your master S3 credentials, Polaris issues short-lived tokens scoped only to the specific table being queried. AnalyticsTeam's Spark cluster gets a token that can only read s3://data/finance/StudentLoans/. Nothing else. The token expires in one hour. The contract still lives in Git. Polaris still enforces it. The mechanism is just different from Unity.

<img width="442" height="405" alt="image" src="https://github.com/user-attachments/assets/c4359f40-8d07-4e9d-8402-72f47c008069" />

```
Data Contract (YAML in Git)
  ← agreed on in a PR, reviewed like code
  ← version-controlled, change-logged
            │
            ▼
Catalog (Unity Catalog or Polaris)
  ← registers the table, stores its metadata
  ← enforces masking, row filters, freshness
  ← logs every access for audit
            │
            ▼
Consumer queries (Spark, Trino, Tableau, Snowflake)
  ← blocked, masked, or filtered automatically
  ← no manual enforcement needed
```

**Summary**
Data contracts are the policy — written by humans, reviewed in Git, agreed on by teams.
Unity Catalog and Polaris are the enforcement — they read the policy and make it impossible to violate.
You need both. A contract without a catalog is just a document nobody checks. A catalog without a contract is just ad-hoc access control that nobody agreed to.

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
