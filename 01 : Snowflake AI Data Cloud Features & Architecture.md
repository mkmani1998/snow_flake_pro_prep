# SnowPro Core COF-C03 — Domain 1 Ultimate Guide

## Snowflake AI Data Cloud Features & Architecture (31% of the exam)

Roughly 30 of \~100 questions. Master this domain and you are already a third of the way to passing. Docs root: **[https://docs.snowflake.com/en/](https://docs.snowflake.com/en/)**

**How to use this guide:** read a section → do the "Trap check" → finish with the 15 practice questions → re-read the cheat sheet the night before.

---

## 1.1 Snowflake Architecture

### The three layers (memorize what lives where)

| Layer | What it does | Billed as |
| --- | --- | --- |
| **Cloud Services** ("the brain") | Authentication, access control, metadata, query parsing/compilation/optimization, infrastructure management, transaction management | Credits, but only the portion **exceeding 10% of daily warehouse credits** is charged |
| **Compute** ("the muscle") | Virtual warehouses (MPP clusters) that execute queries and DML/loading | Credits per second (60-sec minimum on each resume) |
| **Database Storage** ("the disk") | Data reorganized into compressed, columnar, immutable **micro-partitions** in cloud object storage | Flat rate per TB/month (compressed size) |

- Architecture is a **hybrid of shared-disk** (one central storage) **and shared-nothing** (MPP compute nodes with local cache).
- Compute and storage scale **independently**. Many warehouses can read the same data at once with no contention.
- Snowflake runs on AWS, Azure and GCP; the data sits in the cloud provider's blob storage, managed entirely by Snowflake. You cannot access the files directly.
- Metadata-only operations (e.g., `SELECT COUNT(*)` on a table, `MIN/MAX` on a column, `SHOW`, `DESCRIBE`) are answered from Cloud Services **without a running warehouse**.

### Editions: what each adds (a favorite question)

| Feature | Standard | Enterprise | Business Critical | VPS |
| --- | --- | --- | --- | --- |
| SSO/federated auth, OAuth, MFA, network policies, RBAC | ✔ | ✔ | ✔ | ✔ |
| Automatic encryption, Secure Data Sharing, Snowpipe, Time Travel **1 day** | ✔ | ✔ | ✔ | ✔ |
| **Multi-cluster warehouses** | ✘ | ✔ | ✔ | ✔ |
| **Time Travel up to 90 days** | ✘ | ✔ | ✔ | ✔ |
| **Materialized views** | ✘ | ✔ | ✔ | ✔ |
| **Search Optimization / Query Acceleration** | ✘ | ✔ | ✔ | ✔ |
| **Masking policies, row access policies**, tagging, classification | ✘ | ✔ | ✔ | ✔ |
| Periodic rekeying of encrypted data | ✘ | ✔ | ✔ | ✔ |
| **HIPAA / PCI DSS / HITRUST** support, **Tri-Secret Secure** (customer-managed key), **private connectivity** (PrivateLink), **account failover/failback** | ✘ | ✘ | ✔ | ✔ |
| Dedicated, isolated environment (own metadata store) | ✘ | ✘ | ✘ | ✔ |

**Mnemonic:** Standard = basics. Enterprise = **scale + governance** (multi-cluster, 90-day TT, MVs, masking). Business Critical = **compliance + DR**. VPS = **isolation**.

> **Trap check:** Network policies, SSO and MFA are in *all* editions. Masking policies are *not* in Standard.

---

## 1.2 Interfaces & Tools

| Tool | Use it for |
| --- | --- |
| **Snowsight** | Default web UI: worksheets, dashboards, notebooks, Streamlit apps, Query Profile, admin (Admin → Cost Management, Users & Roles, Marketplace) |
| **Snowflake CLI** (`snow`) | **Modern** command-line tool for developers and **CI/CD**: `snow sql`, `snow streamlit`, `snow app` (Native Apps), `snow snowpark`, `snow git`. Config in `config.toml` (connections) |
| **SnowSQL** | **Legacy** CLI, still used for scripting, `PUT`/`GET` to internal stages |
| **IDE integration** | **Snowflake extension for VS Code**: run SQL, manage connections, Snowpark, Native App development |
| **Drivers/connectors** | Python, JDBC, ODBC, Node.js, Go, .NET, PHP; **SQL API** (REST) for programmatic access (Domain 3.3) |

> **Trap check:** If the scenario says *DevOps, automation, deploy an app, CI/CD pipeline* → **Snowflake CLI**. `PUT`/`GET` cannot be run from Snowsight worksheets (use CLI/SnowSQL/drivers).

Docs: `/en/user-guide/snowsight-gs`, `/en/developer-guide/snowflake-cli/index`

---

## 1.3 Object Hierarchy, Parameters & Context

### Hierarchy

```
Organization  (ORGADMIN)
 └─ Account   (ACCOUNTADMIN)
     ├─ Account-level objects: users, roles, warehouses, databases, shares,
     │   resource monitors, network policies, integrations, replication/failover groups,
     │   applications (installed Native Apps)
     └─ Database
         ├─ Database roles
         └─ Schema
             └─ Schema-level objects: tables, views, stages, file formats, pipes, streams,
                tasks, sequences, UDFs, stored procedures, dynamic tables, tags, policies,
                ML models, alerts, secrets
```

- Fully qualified name: `database.schema.object`.
- **Warehouses, users, roles, shares, resource monitors are account-level**, not inside a database.
- Pipes, streams, tasks, stages, file formats, sequences live **in a schema**.

### Parameters

Three types:

1. **Account parameters** (set only at account level, e.g., `NETWORK_POLICY`, `PERIODIC_DATA_REKEYING`)
2. **Session parameters** (can be set at account → user → session, e.g., `TIMEZONE`, `QUERY_TAG`, `STATEMENT_TIMEOUT_IN_SECONDS`)
3. **Object parameters** (set on account or the object, e.g., `DATA_RETENTION_TIME_IN_DAYS`, `MAX_CONCURRENCY_LEVEL`)

**Precedence rule:** the more specific level **overrides** the broader one.

- Session parameters: **Session > User > Account** (Snowflake defaults sit below all).
- Object parameters: **Object (table) > Schema > Database > Account**. A table with no explicit value inherits from its schema.
- **Exception:** if `STATEMENT_TIMEOUT_IN_SECONDS` (or the queued-timeout) is set on both the **warehouse and the session**, the **lowest non-zero** value wins.
- View them: `SHOW PARAMETERS [LIKE ...] IN {ACCOUNT | USER u | SESSION | WAREHOUSE w | TABLE t}`.

### Session & context variables

- **Context functions:** `CURRENT_ROLE()`, `CURRENT_USER()`, `CURRENT_WAREHOUSE()`, `CURRENT_DATABASE()`, `CURRENT_SCHEMA()`, `CURRENT_SESSION()`, `CURRENT_ACCOUNT()`, `CURRENT_REGION()`.
- **Session variables:** `SET v = 'x';` → use as `$v` or `IDENTIFIER($v)` for object names; `UNSET v;`; `SHOW VARIABLES;`. They live only for the session.
- Set context: `USE ROLE`, `USE WAREHOUSE`, `USE DATABASE`, `USE SCHEMA`, `USE SECONDARY ROLES`.

> **Trap check:** `DATA_RETENTION_TIME_IN_DAYS` is set at account = 10, schema = 3, table unset → the table has **3**.

---

## 1.4 Virtual Warehouses (the most-tested topic in Domain 1)

### Sizes & credits (doubling rule)

| XS | S | M | L | XL | 2XL | 3XL | 4XL | 5XL | 6XL |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2 | 4 | 8 | 16 | 32 | 64 | 128 | 256 | 512 |

*Credits per hour.* Each size up **doubles** compute and cost. A job that runs in half the time on the next size costs about the same. Example: a Large running 30 min = 8 × 0.5 = **4 credits**.

### Types

| Type | Notes |
| --- | --- |
| **Standard Gen 1** | Original general-purpose warehouse; SQL, loading, BI |
| **Standard Gen 2** | Newer hardware; faster for many scans/DML; **not always cheaper**, so test both. Not available in every region; **not for Snowpark-optimized**; no 5XL/6XL |
| **Snowpark-optimized** | Much more memory per node; for ML training, large Snowpark UDFs/procs, memory-hungry work; **Medium or larger only** (no XS/S) |
| Default warehouse for Notebooks | Listed in the guide but **not tested until globally GA** |

### Key properties

- `AUTO_SUSPEND` (seconds; SQL default 600; `0`/`NULL` = never), `AUTO_RESUME` (default TRUE), `INITIALLY_SUSPENDED`.
- **Billing:** per second, **60-second minimum** every time the warehouse starts/resumes. Suspended = no compute credits.
- **Resizing** applies to *new* queries; running queries are unaffected. Increasing size can also help queued queries start.
- **Suspending clears the local SSD cache** (warehouse cache).
- Concurrency controls: `MAX_CONCURRENCY_LEVEL` (default 8), `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS`, `STATEMENT_TIMEOUT_IN_SECONDS` (default 2 days).

### Scale UP vs scale OUT (never mix these up)

| Problem | Fix | Mechanism |
| --- | --- | --- |
| One query is slow / **spills to disk** / complex joins, big sorts | **Scale up** (bigger size) | Resize warehouse (vertical) |
| Many users → **queuing** | **Scale out** (more clusters) | Multi-cluster warehouse (horizontal; Enterprise+) |
| Different teams/workloads interfere | **Separate warehouses** | One per team/workload (works in every edition) |

> Scaling out does **not** make a single query faster. Scaling up does **not** fix concurrency well.

### Multi-cluster warehouses (Enterprise+)

- `MIN_CLUSTER_COUNT` / `MAX_CLUSTER_COUNT`.
  - **Maximized** mode: MIN = MAX → all clusters always running.
  - **Auto-scale** mode: MIN \< MAX → Snowflake starts/stops clusters.
- **Scaling policy** (auto-scale only):

|  | **STANDARD** (default) | **ECONOMY** |
| --- | --- | --- |
| Goal | **Minimize queuing** | **Conserve credits** |
| Starts a new cluster | When queries queue / the load exceeds capacity | Only if there's enough work to keep it busy for about **6 minutes** |
| Shuts a cluster down | After **2–3** consecutive low-load checks (1-min intervals) | After **5–6** consecutive checks |
| Trade-off | Higher cost, low wait | Lower cost, queries may wait |

### Warehouse configuration by use case

| Use case | Recommended setup |
| --- | --- |
| **Ad-hoc / exploratory** | Small–Medium, **short auto-suspend** (60 s), auto-resume on; scale up if queries are heavy |
| **Data loading** | Size by **number of files, not size of one file** (each file loads on one thread; \~100–250 MB compressed files). XS–M usually enough; **separate from BI** |
| **BI / reporting (many users)** | **Multi-cluster, auto-scale**; consider a **longer auto-suspend** to keep the warehouse cache warm |
| **Complex queries / heavy transforms** | **Scale up** (larger size); watch for **spilling** |
| **Different teams** | **Separate warehouses** per team; use resource monitors/tags for chargeback |
| **High concurrency** | Multi-cluster (Standard policy for latency-sensitive dashboards) |
| **Memory-intensive ML** | **Snowpark-optimized** (Medium+) |

> **Trap check (classic):** auto-suspend too short on a BI warehouse = warehouse cache is lost, so repeated dashboard queries re-read from remote storage. Trade-off: cache retention vs idle credits.

Docs: `/en/user-guide/warehouses-overview`, `/en/user-guide/warehouses-multicluster`, `/en/user-guide/warehouses-considerations`

---

## 1.5 Storage Concepts

### Micro-partitions

- **50–500 MB** (uncompressed), contiguous, **columnar**, compressed, **immutable** (DML creates new partitions).
- Created **automatically** in insertion order; no manual partitioning or indexes.
- Metadata per partition: **min/max per column, distinct counts, null counts** → enables **partition pruning**.
- Query `FROM t WHERE date = ...` skips partitions whose min/max can't match.

### Clustering

- **Natural clustering** = load order. **Clustering key** = user-defined columns to co-locate similar values.
- Use for **very large (multi-TB) tables**, frequently filtered/joined on the same columns, with poor natural order.
- Choose ≤ 3–4 columns; put the **lower-cardinality column first**; avoid extremely high cardinality (e.g., raw timestamps → truncate to date) or too low (booleans).
- **Automatic Clustering** is serverless (charges credits) and re-clusters after DML. Inspect with `SYSTEM$CLUSTERING_INFORMATION` / `SYSTEM$CLUSTERING_DEPTH` (lower depth = better).
- Clustering is **not** for small tables and not a substitute for a right-sized warehouse.

### Table types (must-know table)

| Type | Time Travel | Fail-safe | Persistence / notes |
| --- | --- | --- | --- |
| **Permanent** (default) | 0–1 day (Std), **0–90 (Ent+)** | **7 days** | Until dropped |
| **Transient** | 0–**1** day | **None** | Until dropped; cheaper; ideal for staging/ETL scratch. Also transient databases/schemas |
| **Temporary** | 0–**1** day | **None** | **Session only**; invisible to other sessions; dropped at session end; can shadow a same-named permanent table |
| **External** | ✘ | ✘ | Data stays in external stage; **read-only**; columns from a `VALUE` variant column; metadata refreshed manually or via event notifications |
| **Apache Iceberg™** | Governed by the catalog | ✘ | Open table format (Parquet + metadata) in **customer-owned external volume**; **Snowflake-managed catalog** = full read/write; externally-managed catalog = mostly read |
| **Dynamic** | Yes | Yes | Declarative pipeline: query + **`TARGET_LAG`** + warehouse; Snowflake auto-refreshes (incremental or full) |

> Fail-safe = 7 days, **Snowflake-only** recovery, **permanent tables only**, starts after Time Travel ends. Storage costs apply to both.

### View types

| View | Key facts |
| --- | --- |
| **Standard** | Saved query, no stored data, always current, definition visible to those with privileges |
| **Secure** | **Definition hidden** from non-owners; bypasses some optimizations that could leak data; **required for sharing** views; slightly slower |
| **Materialized** | **Stored results**, auto-maintained by serverless background process (credits); **Enterprise+**; **single table only** (no joins), limited aggregate/functions; great for repeated, expensive aggregations on slowly changing data |

Choosing: joins/complex transforms needing auto-refresh → **Dynamic table**. Single-table pre-aggregation → **Materialized view**. Hide logic/share safely → **Secure view**.

Docs: `/en/user-guide/tables-micro-partitions`, `/en/user-guide/tables-clustering-keys`, `/en/user-guide/tables-temp-transient`, `/en/user-guide/views-introduction`

---

## 1.6 AI/ML & Application Development

| Feature | What it is | Remember |
| --- | --- | --- |
| **Snowflake Notebooks** | Cell-based Python/SQL/Markdown in Snowsight | Runs on a warehouse or container runtime |
| **Streamlit in Snowflake** | Build & host Python data apps inside Snowflake | Data never leaves Snowflake; runs on a warehouse; governed by RBAC |
| **Snowpark** | DataFrame API (Python, Java, Scala) executed **inside Snowflake** | **Lazy evaluation**: DataFrames become SQL and run only on an *action* (`collect`, `show`, `to_pandas`). Also UDFs, UDTFs, stored procedures. Use Snowpark-optimized WH for heavy memory |
| **Cortex AI SQL functions** | LLM functions callable from SQL (e.g., `AI_COMPLETE`, `AI_CLASSIFY`, `AI_EXTRACT`, `AI_SENTIMENT`, `AI_TRANSLATE`, `AI_EMBED`; older `SNOWFLAKE.CORTEX.*` names) | Serverless, billed per token; data stays in the Snowflake perimeter |
| **Cortex Search** | Managed **hybrid (vector + keyword) search** over text | Use for **RAG** and unstructured document search |
| **Cortex Analyst** | **Natural language → SQL** over **structured** data | Driven by a **semantic model** (YAML) / semantic view; business users ask questions |
| **Snowflake ML** | End-to-end ML: **Feature Store**, **Model Registry**, ML functions (forecasting, anomaly detection, classification) | Models are schema-level objects |

> **Trap check:** *Structured data + English questions* → Cortex Analyst. *Unstructured docs + retrieval/RAG* → Cortex Search. *Run an LLM prompt per row in SQL* → AI SQL functions.

Docs: `/en/user-guide/snowflake-cortex/aisql`, `/en/developer-guide/snowpark/index`, `/en/developer-guide/streamlit/about-streamlit`

---

## Practice Questions (with explanations)

**1.** A team on **Standard Edition** has queries queuing at peak from hundreds of dashboard users. What is the right fix? A. Set scaling policy to Economy · B. Create a multi-cluster warehouse · C. Upgrade to Enterprise or higher to use multi-cluster, or split users across separate warehouses · D. Enable Search Optimization **Answer: C.** Multi-cluster needs Enterprise+. Separate warehouses work on any edition.

**2.** One nightly query joining huge tables shows **"Bytes spilled to remote storage."** Best action? A. Add clusters · B. **Increase warehouse size** · C. Lower auto-suspend · D. Switch policy to Standard **Answer: B.** Spilling = not enough memory → scale up.

**3.** BI dashboards hit the same tables all day but the first query after every pause is slow. Auto-suspend is 60 s. What helps? **Answer:** Increase auto-suspend so the **warehouse cache** isn't discarded between bursts (accept some extra idle credits).

**4.** Which layer compiles and optimizes a query? **Answer:** **Cloud Services.** Execution happens in Compute.

**5.** `DATA_RETENTION_TIME_IN_DAYS`: account = 10, schema `S` = 3, table `T` in `S` has no setting. What is T's retention? **Answer:** **3** (object-level inheritance: table > schema > database > account).

**6.** Session `STATEMENT_TIMEOUT_IN_SECONDS` = 3600; the warehouse has 600. Which applies? **Answer:** **600**. The lowest non-zero value wins.

**7.** Need a staging table that persists across sessions, has no Fail-safe cost, and needs at most 1 day of Time Travel. **Answer:** **Transient table.** (Temporary would vanish at session end.)

**8.** Multi-select: Which require **Enterprise Edition or higher**? (a) network policies (b) multi-cluster warehouses (c) materialized views (d) SSO (e) 90-day Time Travel **Answer: b, c, e.**

**9.** You must pre-compute an aggregation over a **join** of two tables and keep it auto-refreshed. Choose the object. **Answer:** **Dynamic table.** Materialized views can't include joins.

**10.** You want to share a view with another account without exposing its definition. **Answer:** **Secure view.**

**11.** A dashboard warehouse is a multi-cluster auto-scale. Analysts accept a few seconds of waiting; the priority is the fewest running clusters. Which policy? **Answer:** **Economy.**

**12.** A Snowpark DataFrame chain of `filter().select()` runs instantly with no results. Why? **Answer:** **Lazy evaluation**; nothing executes until an action (`collect`, `show`).

**13.** Business users want to ask "revenue by region last quarter?" in plain English against tables. **Answer:** **Cortex Analyst** (with a semantic model).

**14.** A GitHub Actions pipeline must deploy a Streamlit app and Native App to Snowflake. **Answer:** **Snowflake CLI** (`snow streamlit deploy`, `snow app`).

**15.** A Large warehouse runs 90 seconds, then suspends. How many credits are billed? **Answer:** Large = 8 credits/hr; 90 s > the 60-s minimum → 8 × 90/3600 = **0.2 credits**.

---

## Domain 1 Cheat Sheet (read the night before)

- **Layers:** Cloud Services = brain (auth, metadata, optimization). Compute = warehouses. Storage = micro-partitions.
- **Editions:** Multi-cluster, 90-day TT, MVs, masking, Search Opt, Query Accel = **Enterprise+**. HIPAA/PCI, Tri-Secret, PrivateLink, failover = **Business Critical**. SSO/MFA/network policies = **all**.
- **CLI:** `snow` = modern/CI-CD; SnowSQL = legacy.
- **Objects:** warehouses, users, roles, shares, resource monitors = account-level; pipes, streams, tasks, stages = schema-level.
- **Parameters:** more specific level wins; statement timeout on warehouse + session → lowest non-zero wins.
- **Warehouses:** size doubles credits; **per-second, 60-s min**; resize affects new queries; suspend clears cache; **up = complexity/spill, out = concurrency**.
- **Scaling policy:** Standard = no queue (2–3 checks); Economy = full clusters (\~6 min of work, 5–6 checks).
- **Micro-partitions:** 50–500 MB, immutable, columnar, min/max metadata → pruning.
- **Tables:** Permanent (7-day FS) · Transient (no FS, ≤1d TT) · Temporary (session, no FS) · External (read-only) · Iceberg (open, external volume) · Dynamic (TARGET_LAG).
- **Views:** Standard · Secure (hidden, shareable) · Materialized (single table, Ent+).
- **AI:** Analyst = NL→SQL (structured). Search = hybrid RAG search. AI SQL = LLM per row. Snowpark = lazy DataFrames. Snowflake ML = registry + feature store.

## 7-Day Domain 1 Plan

| Day | Focus | Hands-on |
| --- | --- | --- |
| 1 | 1.1 Architecture + editions | Trial account, explore Snowsight |
| 2 | 1.2 + 1.3 | Install `snow` CLI; `SHOW PARAMETERS`; `SET`/`$var` |
| 3 | 1.4 warehouses | Create multi-cluster WH; compare Standard vs Economy config; resize |
| 4 | 1.5 storage | Create transient/temp/dynamic tables; `SYSTEM$CLUSTERING_INFORMATION` |
| 5 | 1.5 views + 1.6 AI | Secure vs materialized view; try an AI SQL function, a notebook |
| 6 | Practice questions + weak spots | Re-do questions without notes |
| 7 | Cheat sheet + official practice exam (Domain 1 items) | Review misses |

Official reference links live in your uploaded study guide (Domain 1.0 Study Resources) and at **[https://docs.snowflake.com/en/](https://docs.snowflake.com/en/)**.
