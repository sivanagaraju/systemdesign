# 02 - CDC System Design & Deep Dives

> **Moving beyond "It works" to "It scales and handles failure."**

At a Staff+ level, understanding *how* CDC works is insufficient. You must understand *how it breaks* and *how to guarantee correctness*.

---

## 1. Ordering Guarantees (The "Sequence" Problem)

In distributed systems, preserving the order of events is the hardest problem. In CDC, it is **non-negotiable**.

### The Problem
If a `CREATE` event and an `UPDATE` event for the same row arrive out of order, your destination tries to update a row that doesn't exist yet.

### Solution Patterns
*   **Partitioning by Key**: Ensure all events for a specific Primary Key (e.g., `User_ID: 123`) always go to the *same* Kafka partition / Shard.
    *   *Why?* Kafka guarantees ordering *within* a partition, but not across partitions.
*   **Sequence Numbers (LSN/SCN)**: Every CDC event carries a `Log Sequence Number` (LSN) from the source DB.
    *   *Sink Logic*: `if event.LSN < target.current_LSN: ignore` (Idempotency).

---

## 2. The "Initial Load" Problem (Snapshotting)

How do you start CDC on a 10TB table without stopping the database?

### The Naive Approach
1.  Stop writes.
2.  Dump database.
3.  Start CDC.
4.  Resume writes.
*   *Verdict*: **Unacceptable** for 24/7 systems.

### The "Incremental Snapshot" Pattern (Debezium / Netflix DBLog)
This is a Staff-level design pattern designed to perform lock-free snapshots.

1.  **Chunking**: Divide the table into chunks by Primary Key.
2.  **Watermarking**: For each chunk:
    *   Record current LSN positions (`Low Watermark`).
    *   Read the chunk.
    *   Record current LSN again (`High Watermark`).
3.  **Deduplication Window**: Any CDC events that occurred between `Low` and `High` watermarks are reconciled with the read chunk to ensure consistency.
4.  **Interleaving**: Snapshot reads are interleaved with streaming events. The pipeline never stops.

---

## 3. Handling Schema Drift

What happens when an upstream engineer runs `ALTER TABLE Users ADD Column Age INT`?

### Failure Modes
1.  **Broken Pipeline**: The CDC parser fails because the binary log format changed.
2.  **Data Loss**: The new column is ignored and not propagated to the Data Lake.
3.  **Poison Pills**: The consumer crashes trying to process the message with unexpected fields.

### Design Solutions
*   **Schema Registry**: Use a central registry (e.g., Confluent Schema Registry) to version schemas.
    *   *Producer*: Registers new schema version ID with the message.
    *   *Consumer*: Fetches new schema to deserialize.
*   **Evolutionary Rules**:
    *   *Backward Compatibility*: New fields have default values. Old consumers can ignore new fields.
    *   *Forward Compatibility*: Consumers can read data written by producers with older schemas.

---

## 4. Fault Tolerance & Checkpointing

CDC is a long-running process. It *will* crash. How do we resume?

### Offset Management
*   **Source-side**: The database manages the replication slot (e.g., Postgres). If the consumer doesn't acknowledge (ACK) the LSN, the DB keeps the logs.
    *   *Risk*: If consumer is down for too long, WAL files fill up usage disk space -> **Database Crash**.
*   **Sink-side**: The consumer (e.g., Kafka Connect) stores the last processed offset.
    *   *Crash Recovery*: On restart, read "Last Offset" -> Request DB to stream from `Last Offset + 1`.

---

## 5. Deduplication & Idempotency

Network retries will cause duplicate messages.

*   **At-Least-Once Delivery**: The standard guarantee. You might get the same update event twice.
*   **Idempotent Sinks**:
    *   *Upsert Semantics*: Instead of `INSERT`, use `MERGE` or `UPSERT`.
    *   *Version Checking*: Only apply update if `incoming_ts > stored_ts`.

---

## 🔭 Deep Dive: The "Hard Delete" Trap

In a Data Lake (Parquet/Delta/Iceberg), you usually perform **Soft Deletes** (append-only architecture).

**Scenario**: User deletes account.
1.  **Source DB**: Row removed. Log: `DELETE ID: 5`.
2.  **CDC Stream**: Propagates `DELETE ID: 5`.
3.  **Data Lake**:
    *   *Bad Design*: You try to physically delete the row from a Parquet file. (Expensive rewrites).
    *   *Good Design (Merge on Read)*: You write a new "Tombstone" record (`ID: 5, Op: DELETE`).
    *   *Read-Time*: The query engine filters out rows where the latest record is a Tombstone.

This is the **LSM-Tree (Log-Structured Merge Tree)** principle applied to Data Lakes.
