# 00 - Architecture Design & Patterns

> **Senior Staff Data Engineer Level**
> Going beyond "how to draw" to "how to design" resilient, scalable, and maintainable data systems.

This module is the **core of your System Design interview**. At a senior/staff level, you are expected to:
1.  **Clarify Constraints** (CAP Theorem, 5 Vs, SLAs).
2.  **Make Trade-offs** (Push vs Pull, Batch vs Stream, Lambda vs Kappa).
3.  **Handle Failure** (Circuit Breakers, Dead Letter Queues, Disaster Recovery).
4.  **Estimate Capacity** (Back-of-the-envelope math).

---

## 📚 Curriculum

### Core Design Patterns (Deep Dives)

| File | Level | Focus | Key Concepts |
|------|-------|-------|--------------|
| [01-lambda-architecture.md](./01-lambda-architecture.md) | **Principal** | **Lambda & Kappa** | Batch Layer vs Speed Layer, Serving Layer merge strategies, Immutable Master Dataset. |
| [02-architecture-design-patterns.md](./02-architecture-design-patterns.md) | **Staff** | **System Design Framework** | CAP Theorem, Ingestion Patterns (Push/Pull), Storage Engines (LSM vs B-Tree), Capacity Planning Math. |
| [03-orchestration-patterns.md](./03-orchestration-patterns.md) | **Senior** | **Workflow Management** | Airflow vs ADF vs Databricks, DAG Design, Idempotency, Backfill Patterns. |
| [04-late-arriving-data.md](./04-late-arriving-data.md) | **Senior** | **Time Handling** | Event Time vs Processing Time, Watermarks, Late Dimension handling. |
| [05-petabyte-scale-patterns.md](./05-petabyte-scale-patterns.md) | **Principal** | **Extreme Scale** | Sharding strategies, Sort-Merge-Bucket Joins, Small Files Problem, Geo-replication. |
| [06-realistic-interview-questions.md](./06-realistic-interview-questions.md) | **All** | **Battle Testing** | 20+ Real Interview Questions with "Staff Level" answers. |

### Critical "Gotcha" Scenarios (The 20% that causes 80% of outages)

| File | Scenario | Deep Dive Concepts |
|------|----------|--------------------|
| [07-out-of-order-events.md](./07-out-of-order-events.md) | **Out-of-Order** | Watermarking internals, State Store management, Trigger policies. |
| [08-duplicate-records.md](./08-duplicate-records.md) | **Deduplication** | Distributed dedupe strategies, `dropDuplicates` internals, Window ranking. |
| [09-schema-drift.md](./09-schema-drift.md) | **Evolution** | Schema Registry patterns, `mergeSchema` risks, Compatibility modes (Backward/Forward). |
| [10-backfill-scenarios.md](./10-backfill-scenarios.md) | **Reprocessing** | Idempotent pipelines, Partition exchange patterns, "Correctness" at scale. |
| [11-cdc-deletes.md](./11-cdc-deletes.md) | **Deletes** | Soft vs Hard Deletes, GDPR compliance patterns, Merge-on-Read vs Copy-on-Write. |
| [12-data-reconciliation.md](./12-data-reconciliation.md) | **Correctness** | Checksum validation, Row count invariant checks, Audit tables. |
| [13-hot-partitions.md](./13-hot-partitions.md) | **Skew** | Salting keys, Broadcast joins, AQE internals. |
| [14-exactly-once-processing.md](./14-exactly-once-processing.md) | **Semantics** | Write-Ahead Logs (WAL), Checkpointing, Two-Phase Commit (2PC) concepts. |

---

## 🎯 The "Staff Engineer" Architecture Template

When asked to design a system, following this mental model:

### 1. Constraints & SLA
-   **RPO/RTO**: How much data can we lose? How fast must we recover?
-   **Freshness**: Sub-second vs Hourly?
-   **Throughput**: Events per second (Peak vs Avg)?

### 2. The 5-Layer Design
```mermaid
graph TB
    subgraph "Layer 1: Ingestion (Buffer)"
        Kafka[Kafka / Event Hub]
        Note1[Decouples Producer/Consumer<br/>Handles Bursts]
    end

    subgraph "Layer 2: Storage (Truth)"
        Bronze[Bronze (Raw/Immutable)]
        Silver[Silver (Clean/Enriched)]
        Gold[Gold (Aggregated)]
    end

    subgraph "Layer 3: Compute (Processing)"
        Stream[Spark Structured Streaming]
        Batch[Spark Batch Daily Jobs]
    end

    subgraph "Layer 4: Serving (Read)"
        API[API / App]
        DW[Synapse / Snowflake]
    end

    subgraph "Layer 5: Governance"
        UC[Unity Catalog]
        Monitor[Observability]
    end

    Kafka --> Stream
    Kafka --> Batch
    Stream --> Bronze
    Batch --> Bronze
    Bronze --> Silver
    Silver --> Gold
    Gold --> DW
```

---

## 🏗️ Common Trade-offs to Discuss

| Pattern A | Pattern B | The Trade-off |
| :--- | :--- | :--- |
| **Micro-batch (Spark)** | **Continuous (Flink)** | Throughput vs Latency. Spark has higher throughput/efficiency; Flink has lower latency. |
| **ETL** | **ELT** | Compute governance. ELT pushes compute to the Warehouse ($$$) but is more flexible. |
| **Lambda** | **Kappa** | Complexity vs Completeness. Lambda handles reprocessing better; Kappa is simpler code. |
| **Strict Schema** | **Schema Evolution** | Data Quality vs Agility. |

---

## 📖 Next Steps

Start with **[02-architecture-design-patterns.md](./02-architecture-design-patterns.md)** to build your framework.
