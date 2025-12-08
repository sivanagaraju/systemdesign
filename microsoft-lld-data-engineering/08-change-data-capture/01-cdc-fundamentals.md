# 01 - Change Data Capture (CDC) Fundamentals

> **"Data is Water": Understanding the Flow of Change**

For a Senior Staff Data Engineer, Change Data Capture (CDC) isn't just about moving data; it's about shifting your mental model from *static state* to *continuous flow*.

## 📝 Formal Definition via System Design Lens

**Change Data Capture (CDC)** is a software design pattern used to determine (and track) the data that has changed so that action can be taken using the changed data.

In a distributed system context, CDC is the mechanism that turns a **Database** (a shared mutable state) into an **Event Stream** (an immutable log of facts). This decoupling allows downstream systems (Data Lakes, Caches, Search Indices) to stay consistent with the source *eventually*, without tight coupling or performance degradation of the operational system.

---

## 🌊 The "Water" Analogy: Static Pond vs. Running River

To explain CDC to stakeholders (or in an interview), use the **Water Analogy**.

### 1. The Static Pond (Batch / Snapshot)
Imagine a pond. To know the state of the water (quality, level, temperature), you have to go to the pond, dip a bucket in, and test it.
*   **Method**: `SELECT * FROM table` (Full Snapshot).
*   **The Problem**: If a fish swam by (a row was inserted) and then left (deleted) between your bucket dips, **you never knew it was there**. You only see the *current state*, not the *history of events*.
*   **Cost**: You have to drain the entire pond (read the whole table) just to check for changes.

### 2. The Running River (CDC / Streams)
Now imagine a river flowing past you. You stand on the bank and watch every drop of water (every event) pass by.
*   **Method**: Reading the Transaction Log (Log-based CDC).
*   **The Power**: You see *everything*. The fish entering, the fish eating, the fish leaving. You have a **complete narrative** of what happened, not just the final result.
*   **Cost**: You must process the flow continuously. If you look away, you might miss something (unless you have a buffer/offset).

---

## 🏗️ CDC Architectures: Pull vs. Push

There are two fundamental ways to capture change. In a system design interview, effectively contrasting these shows seniority.

### 1. Query-Based CDC (The "Pull" Model)
*   **Mechanism**: You actively poll the source database.
*   **Analogy**: Asking your friend "Are you ready?" every 5 seconds.
*   **Patterns**:
    *   **Timestamp-based**: `SELECT * FROM table WHERE modified_at > @last_check`
    *   **Version-based**: `SELECT * FROM table WHERE version_id > @last_version`
    *   **Status-based**: `SELECT * FROM table WHERE status = 'PENDING'`

#### ⚠️ Staff-Level Implications (The "Gotchas")
*   **The "Hard Delete" Problem**: If a row is deleted, it vanishes. Your query `WHERE modified_at > x` returns nothing for that row. You miss the delete event completely. *Mitigation: Soft deletes (`is_active=false`).*
*   **The "Double Update" Problem**: If a row changes twice between your polls (A -> B -> C), you only see C. You missed the transition state B.
*   **Performance penalty**: You are competing for resources (IOPS, CPU) with the operational workload.

### 2. Log-Based CDC (The "Push" Model)
*   **Mechanism**: The database pushes changes as they happen by reading the Write-Ahead Log (WAL) or Transaction Log.
*   **Analogy**: Your friend texting you "I'm ready" exactly when they are done.
*   **Patterns**:
    *   **PostgreSQL**: Write-Ahead Log (WAL) with Logical Decoding plugins (pgoutput, wal2json).
    *   **MySQL**: Binary Log (binlog).
    *   **SQL Server**: Transaction Log (sys.fn_cdc_get_net_changes).
    *   **MongoDB**: Oplog / Change Streams.

#### 🚀 Staff-Level Implications
*   **Zero Impact (mostly)**: Reading the log file (disk I/O) is separate from query execution (engine CPU/Memory). It's asynchronous.
*   **Fidelity**: You see *every* state change. A -> B -> C is captured as two distinct events: `update(A->B)` and `update(B->C)`.
*   **Deletes**: A "Delete" is just another record written to the log. You capture it perfectly.

---

## ⚖️ Trade-off Decision Matrix

When designing a system, use this matrix to justify your choice:

| Feature | Query-Based (Polling) | Log-Based (Streaming) |
| :--- | :--- | :--- |
| **Complexity** | 🟢 Low (Just SQL) | 🔴 High (Requires Kafka/Debezium/Connectors) |
| **Real-time Latency** | 🔴 High (Polling interval dependency) | 🟢 Ms to Seconds (Near Real-time) |
| **Source Impact** | 🔴 High (Table scans/Index lookups) | 🟢 Low (Sequential log reads) |
| **Deletes** | 🔴 Missed (Requires soft-delete) | 🟢 Captured |
| **Schema Changes** | 🟡 Often breaks queries | 🟢 Can carry schema metadata |
| **Permissions** | 🟢 Standard Read access | 🔴 Requires Admin/Replication privileges |

---

## 🧠 Interview Talk Track (The "Senior" Pivot)

> **Interviewer**: "How would you move data from our SQL Server to the Data Lake?"

**Junior Answer**: "I'd write a Python script to select rows where the date is today and save it as Parquet."

**Staff Answer**: "It depends on the **freshness requirements** and the **completeness guarantees** we need.
If this is strictly for nightly reporting and we assume soft-deletes, a **Waterfall/High-Watermark** approach (Query-based) is simplest and cheapest.
However, if we need **event-driven architectures**, need to capture **hard deletes**, or need **intermediate state analysis** (e.g., analyzing the journey of a user's status changes), we must use **Log-based CDC**.
Given Microsoft's ecosystem, I would look at enabling **SQL Server CDC** combined with **Azure Data Factory** or **Debezium** to decouple the source from the sink."
