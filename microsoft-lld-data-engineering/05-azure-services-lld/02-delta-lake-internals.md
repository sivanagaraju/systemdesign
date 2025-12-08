# Delta Lake Internals

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Very Common)

## The Core Question

*"How does Delta Lake provide ACID guarantees on top of object storage?"*

---

## 🏗️ Delta Lake Architecture

```
Delta Table:
┌────────────────────────────────────────────────────────────┐
│  /delta/orders/                                            │
│  ├── _delta_log/                    ← TRANSACTION LOG     │
│  │   ├── 00000000000000000000.json  ← Version 0           │
│  │   ├── 00000000000000000001.json  ← Version 1           │
│  │   ├── 00000000000000000002.json  ← Version 2           │
│  │   └── 00000000000000000010.checkpoint.parquet          │
│  ├── part-00000-abc.snappy.parquet  ← Data files          │
│  ├── part-00001-def.snappy.parquet                        │
│  └── part-00002-ghi.snappy.parquet                        │
└────────────────────────────────────────────────────────────┘
```

---

## 📝 Transaction Log (`_delta_log`)

Each JSON file records **actions** that changed the table:

```json
// 00000000000000000001.json
{
  "commitInfo": {
    "timestamp": 1705312000000,
    "operation": "WRITE",
    "operationParameters": {"mode": "Append"}
  }
}
{
  "add": {
    "path": "part-00003-xyz.snappy.parquet",
    "partitionValues": {"date": "2024-01-15"},
    "size": 123456,
    "modificationTime": 1705312000000,
    "dataChange": true,
    "stats": "{\"numRecords\":10000,\"minValues\":{\"id\":1},\"maxValues\":{\"id\":10000}}"
  }
}
```

### Action Types

| Action | Purpose |
|--------|---------|
| `add` | New file added to table |
| `remove` | File logically deleted |
| `txn` | Streaming transaction ID |
| `protocol` | Version compatibility |
| `metaData` | Schema, partitioning changes |

---

## 🔐 ACID Properties

| Property | How Delta Achieves It |
|----------|----------------------|
| **Atomicity** | Write new files, then atomically update log |
| **Consistency** | Schema enforcement before write |
| **Isolation** | Optimistic concurrency with conflict detection |
| **Durability** | Data in Parquet files, log in cloud storage |

### Optimistic Concurrency

```
Transaction 1                    Transaction 2
     │                                │
     ▼                                ▼
Read version 5                   Read version 5
     │                                │
     ▼                                ▼
Write new files                  Write new files
     │                                │
     ▼                                │
Commit as v6 ✓                        │
                                      ▼
                               Try commit as v6
                                      │
                               CONFLICT! v6 exists
                                      │
                                      ▼
                               Re-read v6, check conflicts
                               If disjoint files → commit as v7
                               If conflict → retry or fail
```

---

## ⏰ Time Travel

```python
# Read specific version
df = spark.read.format("delta") \
    .option("versionAsOf", 5) \
    .load("/delta/orders")

# Read at timestamp
df = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-15 10:00:00") \
    .load("/delta/orders")

# View history
spark.sql("DESCRIBE HISTORY delta.`/delta/orders`")
```

---

## 🧹 Maintenance Operations

```sql
-- Compact small files
OPTIMIZE orders WHERE date >= '2024-01-01';

-- Z-Order for query optimization  
OPTIMIZE orders ZORDER BY (customer_id);

-- Remove old versions (default: 7 days)
VACUUM orders RETAIN 168 HOURS;

-- Purge deleted files immediately (dangerous!)
VACUUM orders RETAIN 0 HOURS;  -- Requires special setting
```

---

## 🎯 Interview Answer

> *"Delta Lake uses a transaction log (`_delta_log`) to provide ACID:*
> - *Each commit creates a JSON file recording add/remove actions*
> - *Readers reconstruct table state by replaying log*
> - *Writers use optimistic concurrency with conflict detection*
> - *Data files are immutable Parquet; deletes are logical (remove action)*
> - *Time travel by reading log at specific version"*

---

## 📖 Next Topic

Continue to [Synapse Distribution Strategies](./03-synapse-distribution-strategies.md).
