# File Formats Deep Dive

> **Interview Frequency:** ⭐⭐⭐⭐ (Common at Microsoft)

## The Core Question

*"Why would you use Parquet over Avro? When would you choose Delta Lake?"*

Understanding file formats shows you know **storage-level optimization**.

---

## 📊 Quick Comparison

| Format | Storage | Best For | Schema Evolution | ACID |
|--------|---------|----------|------------------|------|
| **Parquet** | Columnar | Analytics (read-heavy) | Limited | ❌ |
| **Avro** | Row-based | Streaming, write-heavy | Excellent | ❌ |
| **ORC** | Columnar | Hive ecosystem | Good | ❌ |
| **Delta** | Columnar + Log | Lakehouse (ACID needed) | Excellent | ✅ |

---

## 📦 Parquet

### What Is It?

**Columnar** storage format optimized for analytics workloads.

```
ROW-BASED (CSV):              COLUMNAR (Parquet):
┌────┬─────┬────────┐         ┌────────────────────┐
│ id │name │ amount │         │ id: [1, 2, 3, 4]   │
├────┼─────┼────────┤         ├────────────────────┤
│ 1  │John │ 100    │         │ name: ["John",     │
│ 2  │Jane │ 200    │         │        "Jane",     │
│ 3  │Bob  │ 150    │         │        "Bob",      │
│ 4  │Alice│ 300    │         │        "Alice"]    │
└────┴─────┴────────┘         ├────────────────────┤
                              │ amount: [100, 200, │
                              │          150, 300] │
                              └────────────────────┘
```

### Why Columnar Is Better for Analytics

```python
# Query: Get total amount
SELECT SUM(amount) FROM orders

# Row-based: Must read entire file (all columns)
# Columnar:  Only reads 'amount' column!
```

### Parquet Internal Structure

```
Parquet File:
┌──────────────────────────────────────────┐
│ Row Group 1 (e.g., 128MB of data)        │
│ ┌──────────────────────────────────────┐ │
│ │ Column Chunk: id                     │ │
│ │   └── Page 1, Page 2, ...            │ │
│ ├──────────────────────────────────────┤ │
│ │ Column Chunk: name                   │ │
│ │   └── Page 1, Page 2, ...            │ │
│ ├──────────────────────────────────────┤ │
│ │ Column Chunk: amount                 │ │
│ │   └── Page 1, Page 2, ...            │ │
│ └──────────────────────────────────────┘ │
├──────────────────────────────────────────┤
│ Row Group 2                              │
│ └── ...                                  │
├──────────────────────────────────────────┤
│ Footer (schema, statistics, locations)   │
└──────────────────────────────────────────┘
```

### Predicate Pushdown with Parquet

Each column chunk stores **min/max statistics**:

```python
# Query:
df.filter(df.amount > 250)

# Parquet can skip Row Group 1:
#   Row Group 1: amount min=100, max=200  → SKIP
#   Row Group 2: amount min=150, max=500  → READ
```

### Creating Parquet

```python
# PySpark
df.write.format("parquet").save("/path/to/output")

# With compression
df.write \
    .option("compression", "snappy") \  # or gzip, zstd
    .parquet("/path/to/output")
```

---

## 📄 Avro

### What Is It?

**Row-based** format with excellent schema evolution support.

### When to Use Avro

| Use Case | Why Avro |
|----------|----------|
| **Kafka messages** | Compact, fast serialization |
| **Schema evolution** | Add/remove fields safely |
| **Write-heavy** | Row-based = faster writes |
| **Streaming** | Low latency serialization |

### Schema Evolution Example

```json
// Version 1
{
  "name": "Customer",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"}
  ]
}

// Version 2 (added field with default)
{
  "name": "Customer",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}
  ]
}
// Old data can be read with new schema (email = null)
```

### Creating Avro

```python
df.write.format("avro").save("/path/to/output")
```

---

## 🔷 Delta Lake

### What Is It?

**Parquet + Transaction Log** = ACID-compliant data lake.

```
Delta Table Structure:
/my_delta_table/
├── _delta_log/                    ← Transaction log!
│   ├── 00000000000000000000.json  ← Version 0
│   ├── 00000000000000000001.json  ← Version 1
│   ├── 00000000000000000002.json  ← Version 2
│   └── ...
├── part-00000-abc.snappy.parquet  ← Actual data
├── part-00001-def.snappy.parquet
└── part-00002-ghi.snappy.parquet
```

### The Transaction Log (`_delta_log`)

Each JSON file records **what changed**:

```json
// 00000000000000000001.json
{
  "add": {
    "path": "part-00003-xyz.snappy.parquet",
    "size": 123456,
    "partitionValues": {"date": "2024-01-15"},
    "modificationTime": 1705312000000,
    "dataChange": true
  }
}

// 00000000000000000002.json
{
  "remove": {
    "path": "part-00001-def.snappy.parquet",
    "deletionTimestamp": 1705315600000,
    "dataChange": true
  }
}
```

### ACID Properties

| Property | How Delta Achieves It |
|----------|----------------------|
| **Atomicity** | Write new files, then update log atomically |
| **Consistency** | Schema enforcement before writes |
| **Isolation** | Optimistic concurrency (conflict detection) |
| **Durability** | Data in Parquet files, log in cloud storage |

### Key Delta Features

```python
# 1. Time Travel
df = spark.read.format("delta") \
    .option("versionAsOf", 5) \       # Read version 5
    .load("/path/to/table")

df = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-15") \
    .load("/path/to/table")

# 2. MERGE (Upsert)
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "/path/to/table")
delta_table.alias("target").merge(
    updates_df.alias("source"),
    "target.id = source.id"
).whenMatchedUpdate(set={"value": "source.value"}) \
 .whenNotMatchedInsert(values={"id": "source.id", "value": "source.value"}) \
 .execute()

# 3. Schema Evolution
df.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .save("/path/to/table")

# 4. OPTIMIZE (Compaction)
spark.sql("OPTIMIZE my_table")
spark.sql("OPTIMIZE my_table ZORDER BY (customer_id)")
```

---

## ⚖️ Decision Matrix

```
┌─────────────────────────────────────────────────────────────┐
│                 What's your primary need?                    │
└─────────────────────────────┬───────────────────────────────┘
                              │
      ┌───────────────────────┼───────────────────────┐
      │                       │                       │
      ▼                       ▼                       ▼
 Analytics/BI           Streaming/Kafka         ACID/Updates
 (read-heavy)           (schema evolution)      (upserts needed)
      │                       │                       │
      ▼                       ▼                       ▼
  ┌───────┐               ┌───────┐              ┌───────┐
  │Parquet│               │ Avro  │              │ Delta │
  └───────┘               └───────┘              └───────┘
```

---

## 🎯 Interview Answer

When asked "Parquet vs Avro vs Delta":

> **Parquet:**
> *"Columnar format, best for analytics. Supports predicate pushdown and column pruning. Use when workload is read-heavy and you're querying specific columns."*

> **Avro:**
> *"Row-based with excellent schema evolution. Best for streaming (Kafka) and write-heavy workloads. Schema is embedded, making it self-describing."*

> **Delta:**
> *"Parquet + transaction log for ACID guarantees. Essential when you need MERGE/upserts, time travel, or concurrent writes. Standard at Microsoft for data lakes."*

---

## 📖 Next Topic

Continue to [Debugging OOM Errors](./05-debugging-oom-errors.md) for troubleshooting memory issues.
