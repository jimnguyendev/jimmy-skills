# Architecture Strategy

## Architecture Comparison Matrix

| Dimension | RDW | Data Lake | Modern DW (MDW) | Data Fabric | Data Lakehouse | Data Mesh |
|-----------|-----|-----------|-----------------|-------------|----------------|-----------|
| **Year introduced** | 1984 | 2010 | 2011 | 2016 | 2020 | 2019 |
| **Schema type** | Write | Read | Read + Write | Read + Write | Read | Domain-specific |
| **Centralization** | Centralized | Centralized | Centralized | Centralized | Centralized | Decentralized |
| **Security level** | High | Low-Medium | Medium-High | High | Medium | Domain-specific |
| **Time-to-value** | Medium | Low | Low | Low | Medium | High |
| **Cost** | High | Low | Medium | Medium-High | Low-Medium | High |
| **Difficulty** | Low | Medium | Medium | Medium | Medium-High | High |
| **Maturity** | High | Medium | Medium-High | Medium-High | Medium-High | Low |
| **Best for** | Structured BI, compliance | Raw staging, multi-format ingest | Lake + relational serving | Enterprise governance + metadata | Unified analytics + ML | Decentralized domain ownership at scale |

## Schema Strategy

### Schema-on-write vs schema-on-read

| | Schema-on-write | Schema-on-read |
|-|----------------|----------------|
| **When schema is defined** | At write time | At query time |
| **Advantages** | Clean data, consistent, fast queries | Flexible, ingest any format |
| **Disadvantages** | Upfront work required, less flexible | Complex queries, easy to accumulate dirty data |
| **Used in** | RDW, MDW (serving layer) | Data Lake, Lakehouse |

### Compatibility modes

| Mode | Rule | Use when |
|------|------|----------|
| **Backward** | New schema can read old data | Consumer upgrades lag behind producers |
| **Forward** | Old schema can read new data | Producers evolve faster than consumers |
| **Full** | Both directions work | Critical shared datasets with many consumers |

Default to schema-on-write when business metrics must be trustworthy early. Use schema-on-read only when format diversity or ingestion speed is the binding constraint.

## Detailed Architecture Profiles

### 1. Relational Data Warehouse (RDW)

A relational database optimized for analytics and BI rather than transactional operations. Compute engine and relational storage are tied together. Schema-on-write enforces data quality at ingestion.

**Key characteristics:** OLAP-optimized, schema enforcement, transaction support, audit trails, mature tooling.

**When to choose:** Structured analytics with well-known schemas. Teams that need strong security (row-level, column-level). Analysts who expect predictable relational access. Data under 1TB with primarily structured sources.

**When NOT to choose:** Semi-structured or unstructured data dominates. Storage and compute need to scale independently. ML workloads need direct file-level access.

**Trade-offs:** High cost because compute and storage are coupled. Limited scalability for large-scale unstructured data. Well-understood failure modes.

**Anti-patterns:** Running heavy analytics queries on the production OLTP database instead of a separate RDW. Over-normalizing the warehouse when a star schema would serve BI better.

### 2. Data Lake

Object storage (S3, Azure Blob, GCS) that accepts any format raw. No compute engine included; bring your own processing. Schema-on-read means no upfront schema definition.

**Key characteristics:** Cheap storage, unlimited scale, accepts structured/semi-structured/unstructured data, schema-on-read.

**When to choose:** Need a low-cost staging area for raw multi-format data. Data scientists need direct file access for exploration. Batch ingestion of event logs, clickstream, or media files.

**When NOT to choose:** Business users need self-service BI (they cannot query a lake directly). Governance and security are primary concerns. You need it to be the sole analytics interface.

**Trade-offs:** Without governance, lakes become swamps. End users cannot self-serve. Query performance is poor without additional processing layers.

**Anti-patterns:** Treating the data lake as the final analytics destination for non-technical users. Ingesting everything with no cataloging or retention policy ("dump and pray"). Companies like Hortonworks and MapR failed trying to position the lake as an all-in-one solution.

### 3. Modern Data Warehouse (MDW)

Combines a data lake for staging, transformation, and ML with an RDW for serving, security, and BI. The most widely adopted architecture for mid-size and larger organizations today.

**Key characteristics:** Dual-system (lake + RDW), data replication from lake to warehouse, supports both schema-on-read and schema-on-write, separation of concerns between data science and business analytics.

**When to choose:** Need both flexibility (ML, unstructured data) and mature access patterns (BI, compliance). Multiple user personas: data scientists working in the lake, business analysts querying the warehouse. Organization is mid-size or larger with budget for two systems.

**When NOT to choose:** Team is too small to manage two systems. All data is structured and a simple RDW is sufficient. Cost of maintaining two copies of data is not justified.

**Trade-offs:** Complexity of managing lake + RDW. Data staleness between lake and warehouse (depends on copy frequency). Need two skill sets (lake admin + RDW admin). Higher total cost than either system alone.

**Anti-patterns:** Building the full MDW before having enough data or use cases to justify it. Skipping the data lake and calling a cloud warehouse "modern" because it is in the cloud.

### 4. Data Fabric

An evolution of MDW with additional layers: advanced security, master data management (MDM), API layer, metadata management, and data lineage.

**Key characteristics:** Enterprise-grade governance, row-level and column-level security, MDM for master data consistency, API exposure of data, metadata cataloging and lineage.

**When to choose:** Enterprise that requires strict governance, compliance, and audit capabilities across many data sources. Multiple business units need consistent master data. Regulatory requirements demand full lineage and access control.

**When NOT to choose:** Teams under 50 engineers. Organization does not have mature data practices yet. Cost and complexity cannot be justified by regulatory or scale requirements.

**Trade-offs:** Highest cost and complexity among centralized architectures. Requires significant upfront investment in governance tooling. Can slow delivery if governance processes are over-engineered.

**Anti-patterns:** Adopting data fabric for "future-proofing" when the team has not outgrown MDW. Adding MDM before there is a concrete master data inconsistency problem.

### 5. Data Lakehouse

Uses a single data-lake-based repository with a transactional storage layer (Delta Lake, Apache Iceberg, or Apache Hudi) to replace the separate RDW. One system handles analytics, ML, and BI.

**Key characteristics:** Single repository (no RDW copy), ACID transactions on object storage, schema enforcement at write, time travel, unified batch and streaming, DML support (INSERT, UPDATE, DELETE, MERGE).

**When to choose:** ML and analytics workloads both need first-class access to the same data. Maintaining two copies (lake + RDW) creates unacceptable cost or consistency issues. Team accepts slightly slower query performance in exchange for architectural simplicity.

**When NOT to choose:** Dashboards require millisecond response times (RDW is still faster). Organization needs mature row-level and column-level security today. End users are non-technical and expect full relational semantics. Heavy concurrent user load requires RDW-grade locking and isolation.

**Trade-offs:** See detailed section below.

**Anti-patterns:** Adopting lakehouse because "it is the future" without having ML workloads. Dropping the RDW and then re-adding a relational serving layer that effectively recreates one.

### 6. Data Mesh

An organizationally decentralized model where each business domain owns its own analytical data, infrastructure, and pipelines. A concept and cultural shift, not a technology. Each domain can use MDW, lakehouse, or data fabric internally.

**Key characteristics:** Domain ownership, data as a product, self-serve data infrastructure platform, federated computational governance.

**When to choose:** 100+ engineers across multiple complex domains. Central IT is a proven bottleneck for data access and quality. Organization is ready for a large cultural and organizational shift. Domains have enough headcount to staff their own data teams.

**When NOT to choose:** SMB or startup. Fewer than 50 engineers. Data volume under 1TB. Centralized architecture still handles the load. Team lacks mature data practices to decentralize.

**Trade-offs:** Highest cost and organizational complexity. Gartner Hype Cycle 2023 placed data mesh near peak of inflated expectations with only 5-20% market penetration. Many companies claiming data mesh are actually running data fabric or lakehouse with multiple workspaces.

**Anti-patterns:** Adopting data mesh as a technology decision rather than an organizational one. Implementing only domain ownership (principle 1) without data-as-product discipline and calling it mesh. Startups pursuing mesh because a blog post made it sound modern.

## Modern Data Warehouse Deep-Dive

### Five data flow stages

| Stage | Role | Details |
|-------|------|---------|
| **1. Ingestion** | Bring data into the system | All data types (structured, semi-structured, unstructured), batch or streaming, on-prem or cloud |
| **2. Storage** | Land in data lake | Cloud object storage (S3, Azure Blob), low cost, unlimited scale, built-in HA/DR/encryption |
| **3. Transformation** | Clean and prepare | Separate storage and compute. Data progresses: raw -> conformed -> cleaned -> presentation |
| **4. Modeling** | Copy into RDW | Create relational model (3NF or star schema) in RDW for performance and usability |
| **5. Visualization** | Analyze and report | Business users consume via BI tools (dashboards, reports) from RDW |

Data scientists can pull from any layer: raw, cleaned, presentation (in the lake), sandbox, or RDW.

### Exceptions to the five-stage flow

**Not all presentation data needs to go to RDW.** Modern BI tools can query the lake directly for some datasets, reducing replication cost.

**Not all source data must pass through the lake first.** Structured reference data (dimension tables, lookup tables) can copy straight from source to RDW, bypassing the lake. This is especially useful during on-prem migration: just redirect existing ETL destination to the cloud RDW. Trade-off: lose the lake as backup and single source of truth, and RDW must handle data cleansing.

### Stepping stones to MDW

Full MDW is expensive and time-consuming. Three intermediate patterns provide value while building toward it:

| Stepping Stone | Pattern | When to use | Trade-off |
|---------------|---------|-------------|-----------|
| **EDW Augmentation** | Keep on-prem EDW, add cloud data lake for big data | Existing EDW cannot handle semi-structured or unstructured data | Need new tools for the lake; does not reduce EDW workload; combining data across systems is complex |
| **Temporary Data Lake + EDW** | Data lake as staging area only, all queries go to EDW | Need to offload transformations from EDW to reduce maintenance windows | Easy to implement; with small changes evolves into full MDW |
| **All-in-One** | Everything in the data lake, no RDW | Startups, prototyping, or primarily technical users (data scientists) | Loses performance, security, and user-friendliness of RDW; precursor to lakehouse |

### SMP vs MPP processing

For data under 1TB, SMP (symmetric multiprocessing) is sufficient and cheaper. MPP (massively parallel processing) is justified only when data volume or query concurrency demands distributed compute. Do not over-invest in MPP for small data.

### Self-service BI as success metric

The goal of MDW maturity is self-service BI: business users drag-and-drop to create reports without IT building each one. IT invests upfront in modeling and metadata so end users are unblocked. This frees the data team from the report-request bottleneck.

## Data Lakehouse Deep-Dive

### Transactional storage layers

| Layer | Origin | Notes |
|-------|--------|-------|
| **Delta Lake** | Databricks | Parquet files + transaction log. DML commands, ACID (per-table), time travel, small file compaction, unified batch + streaming, schema enforcement |
| **Apache Iceberg** | Netflix / Apache | Open table format, multi-engine support (Spark, Trino, Flink), hidden partitioning, schema evolution |
| **Apache Hudi** | Uber / Apache | Upsert-oriented, incremental processing, record-level indexing, timeline-based versioning |

All three add ACID transactions, schema enforcement, and time travel on top of object storage. ACID scope is per-table, not cross-table (unlike RDW which spans multiple tables in a single transaction).

### Six problems lakehouse solves vs MDW

| Problem | In MDW | In Lakehouse |
|---------|--------|--------------|
| **Reliability** | Copying data between lake and RDW can fail, causing inconsistency between reports | Single copy eliminates inconsistency |
| **Data staleness** | RDW data lags behind lake (depends on copy frequency) | Single copy is always current |
| **ML/AI support** | Few ML tools work well on RDW; data scientists prefer lake files | Data scientists access the same repository directly |
| **Total cost** | Two copies of data + copy compute + two admin skill sets | One copy, less compute, fewer skill sets |
| **Data governance** | Two systems with two security models increase risk | One system, one security model |
| **Complexity** | Managing lake + RDW requires specialized skills for each | Fewer systems to manage |

### Trade-offs of dropping the RDW

| Dimension | What you lose | Impact |
|-----------|---------------|--------|
| **Performance** | MPP query optimization, advanced indexing (clustered columnstore, full-text), materialized views, advanced join optimization, caching | Queries are slower; Delta Lake Z-order helps but does not match RDW |
| **Security** | Row-level security, column-level security, data-at-rest encryption, column-level encryption, TDE, dynamic data masking | Must implement security in application layer or wait for lakehouse tooling to mature |
| **Metadata and usability** | Schema-on-write guarantees metadata is always attached and in sync. Lakehouse uses file-folder metadata that can drift or go missing | Biggest gap for end users accustomed to relational schemas |
| **Concurrency** | Advanced locking, isolation levels for many simultaneous users | Lower concurrent query throughput |
| **SQL compatibility** | May require Spark SQL instead of ANSI SQL or T-SQL | Stored procedures, views, and reports may need rewriting |
| **Referential integrity** | No foreign key enforcement in Delta Lake | Application must enforce relationships |

**When the trade-offs do not matter:** If average query latency of 5 seconds is acceptable to end users, the lakehouse is viable. If dashboards need millisecond response, copy data into an RDW serving layer (which effectively recreates MDW).

### Relational serving layer pattern

Because lakehouse storage is schema-on-read and file-folder-based, a serving layer is needed for non-technical users. Options: SQL views on Delta Lake, dataset definitions in reporting tools, Apache Hive tables, or ad hoc SQL queries. Done well, end users do not know they are querying a lakehouse. Risk: metadata can drift between the serving layer and the underlying files.

## Data Mesh Deep-Dive

### Four principles (Dehghani)

| # | Principle | Problem it solves | Solution |
|---|-----------|-------------------|----------|
| 1 | **Domain ownership** | Central IT owns analytical data but does not understand it, causing quality issues | Each business domain (not application) owns its operational and analytical data |
| 2 | **Data as a product** | IT transforms data without domain context, introducing errors | Each domain treats its data as a product with consumers as customers. Data products are discoverable, trustworthy, accessible, secure, understandable. Consumed via data contracts |
| 3 | **Self-serve data infrastructure** | Domain teams cannot each build infra from scratch (duplicate effort, inconsistency) | Central platform team provides APIs and tools for domains to provision infrastructure on demand |
| 4 | **Federated computational governance** | Centralized governance creates bottleneck; fully decentralized governance creates inconsistency | Central team (with domain representatives) defines global standards; domain teams implement and monitor |

Principles 1-2 are decentralized. Principles 3-4 require centralized teams. Data mesh is not fully decentralized.

Minimum for a legitimate data mesh: principles 1 + 2. Without data-as-product discipline, separate domain lakes are just data federation, not mesh.

### Three domain types

| Type | Description | Example |
|------|-------------|---------|
| **Source-aligned** | Analytical data corresponding directly to operational sources, copied and transformed from operational DB | Product search, checkout, payment domains |
| **Aggregated** | Combines data from multiple domains, primarily for performance | P&L domain combining manufacturing + sales |
| **Consumer-aligned** | Data transformed for a specific consumer's needs, simplified model, reduced jargon | Supplier data remodeled for non-supplier teams |

### Three mesh topologies

| Topology | Description | Trade-off |
|----------|-------------|-----------|
| **Type 1** | All domains share the same technology and cloud. Shared physical data lake with per-domain folders | Easiest to manage, best cross-domain query performance, least domain autonomy |
| **Type 2** | Same technology, but each domain has its own data lake | Truly decentralized infrastructure, harder to combine data across domains |
| **Type 3** | Each domain chooses its own technology and cloud provider | "Pure" mesh. Author (Serra) predicts this will never be widely adopted due to extreme complexity |

Agility increases and control decreases from Type 1 to Type 3.

### Concrete adoption criteria

**When to consider:** 100+ engineers, multiple complex business domains with distinct data semantics, central IT is a documented bottleneck for data requests, organization has budget and willingness for cultural transformation.

**When NOT to consider:** SMB or startup, fewer than 50 engineers, data volume under 1TB, centralized architecture still handles scale, no mature data practices to decentralize, problem is technical (not organizational).

### Realistic assessment

Gartner Hype Cycle 2023: near peak of inflated expectations, 5-20% market penetration. Gartner predicted data mesh may become obsolete before reaching plateau, evolving into data fabric. At industry conferences, 75% of audiences knew of data mesh, a few were building one, and none had a production deployment. Many companies claiming data mesh were actually running data fabric or lakehouse with multiple workspaces.

Data warehouse project failures are almost never caused by architecture choice. They are caused by people and process problems. Centralized architectures have handled petabytes since the 1980s.

## MDW vs Lakehouse Decision Guide

### Feature comparison

| Feature | MDW | Lakehouse |
|---------|-----|-----------|
| **Number of repositories** | 2 (lake + RDW) | 1 (lake + transactional layer) |
| **Data consistency** | Risk of drift between lake and RDW copies | Single copy, always consistent |
| **Query performance** | RDW is faster (MPP, advanced indexing, caching, materialized views) | Slower; Z-order and file compaction help but do not match RDW |
| **Security** | Mature (row-level, column-level, TDE, dynamic masking) | Still maturing; gaps in row/column-level security |
| **SQL compatibility** | Full ANSI SQL / T-SQL | May require Spark SQL; no native SQL views in Delta Lake |
| **Referential integrity** | Enforced via foreign keys | Not enforced; application responsibility |
| **Concurrency** | High (advanced locking, isolation levels) | Lower concurrent query throughput |
| **ML/AI support** | Data scientists prefer lake files; RDW is awkward for ML | Direct file access for ML; single system for both |
| **Total cost** | Higher (two copies, two admin skill sets) | Lower (one copy, fewer skill sets) |
| **End-user experience** | Familiar relational model, metadata always in sync | File-folder metadata can drift; needs serving layer |

### Decision criteria

Choose MDW when:

- Dashboards need millisecond response times
- Regulatory compliance requires mature row-level and column-level security
- Most consumers are non-technical business users expecting relational semantics
- Concurrent user count is high
- Organization has budget and skills for two systems

Choose lakehouse when:

- ML and analytics workloads both need first-class data access
- Cost of maintaining two data copies is a binding constraint
- Data staleness between lake and warehouse is unacceptable
- Team accepts 1-5 second query latency for BI
- Primarily technical users or strong serving-layer engineering

## Architecture Evolution Roadmap

### Stage 1-2 team defaults

Start with the simplest centralized architecture that delivers value:

- **Stage 1 (Reactive):** Data scattered in spreadsheets and production DBs. First step: create a single RDW (managed cloud DB or Postgres + dbt).
- **Stage 2 (Informative):** Data centralized, reporting easier. Solidify the RDW. Add basic data quality checks. Target self-service BI.

Do not add a data lake, lakehouse, or mesh at this stage unless a specific forcing function exists.

### Decision triggers for upgrading architecture

| Trigger | Upgrade path |
|---------|-------------|
| Need to ingest semi-structured or unstructured data at scale | Add data lake as staging layer -> move toward MDW |
| ML workloads require direct file access to training data | Add data lake for ML; keep RDW for BI (MDW) or evaluate lakehouse |
| Cost of maintaining two data copies becomes the top budget item | Evaluate lakehouse to consolidate |
| Central data team is documented bottleneck across 3+ domains | Evaluate data mesh (only at 100+ engineer scale) |
| Regulatory or compliance requirements demand lineage and MDM | Evaluate data fabric layers on top of MDW |
| Dashboard response time degrades below user tolerance | If on lakehouse, add relational serving layer or revert to MDW pattern |

### How to stage architecture transitions

1. **Prove value first.** Deliver dashboards or ML results on the current architecture before proposing a migration.
2. **Use stepping stones.** EDW augmentation or temporary data lake patterns let you test the next stage without a full migration.
3. **Migrate incrementally.** Move one domain or dataset at a time. Run old and new in parallel until confidence is high.
4. **Measure before and after.** Track query latency, data freshness, cost, and user satisfaction across the transition.

### Common anti-patterns in architecture evolution

| Anti-pattern | Why it fails |
|-------------|-------------|
| **Jumping to data mesh at 20 engineers** | Not enough people or domains to justify decentralization. Creates overhead without benefit |
| **Adopting lakehouse for FOMO** | Without ML workloads, lakehouse adds complexity (serving layer, weaker security) for no gain |
| **Skipping RDW and going straight to data lake** | Business users cannot self-serve. Lake becomes a swamp |
| **Building full MDW before having data or use cases** | Expensive infrastructure with no consumers. Deliver value first |
| **Calling a cloud warehouse "modern" without a lake** | "Modern" in MDW means lake + RDW, not just cloud deployment |
| **Over-engineering governance before Stage 3** | Data fabric and MDM layers slow delivery when the team has not outgrown basic access controls |
