# Partitioning vs Bucketing

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Critical for Azure/Databricks roles)

## The Core Question

*"How would you partition this data in ADLS Gen2?"*

This is a **low-level design** question testing your understanding of physical data layout optimization.

---

## 📂 What's the Difference?

| Aspect | Partitioning | Bucketing |
|--------|--------------|-----------|
| **Mechanism** | Splits data into **directories** | Splits data into **fixed number of files** |
| **Key Type** | Low-cardinality (date, region) | High-cardinality (user_id, device_id) |
| **Query Benefit** | **Partition pruning** (skip directories) | **Join optimization** (co-located data) |
| **Schema Change** | Easy to add new partitions | Fixed bucket count, hard to change |

---

## 📁 Partitioning

### Concept

Data is **physically organized into directories** based on partition column values.

```
/data/gaming_sessions/
├── year=2024/
│   ├── month=01/
│   │   ├── region=US/
│   │   │   └── part-00001.parquet
│   │   └── region=EU/
│   │       └── part-00001.parquet
│   └── month=02/
│       └── ...
```

### How It Helps (Partition Pruning)

```sql
-- Query: Get US sessions for January 2024
SELECT * FROM gaming_sessions
WHERE year = 2024 AND month = 1 AND region = 'US';

-- Spark only reads: /year=2024/month=01/region=US/
-- Skips ALL other directories!
```

### Creating Partitioned Tables

**Spark SQL:**
```sql
CREATE TABLE gaming_sessions (
    session_id BIGINT,
    player_id STRING,
    duration_sec INT,
    region STRING,
    session_date DATE
)
USING DELTA
PARTITIONED BY (year INT, month INT, region STRING);

-- Insert with partition columns
INSERT INTO gaming_sessions
SELECT *, year(session_date), month(session_date), region
FROM raw_sessions;
```

**PySpark:**
```python
df.write \
    .partitionBy("year", "month", "region") \
    .format("delta") \
    .mode("overwrite") \
    .save("/data/gaming_sessions")
```

---

## ⚠️ The Small Files Problem

### What Is It?

Over-partitioning creates **too many small files**, killing performance:

```
BAD: Partitioning by user_id (high cardinality)
/data/sessions/
├── user_id=user_001/
│   └── part-00001.parquet  (1 KB)  ← Tiny file!
├── user_id=user_002/
│   └── part-00001.parquet  (500 bytes)
├── user_id=user_003/
│   └── part-00001.parquet  (2 KB)
... (millions of tiny files)
```

### Why It's Bad

| Problem | Impact |
|---------|--------|
| **Metadata overhead** | Storage system struggles with millions of files |
| **Read overhead** | Opening each file has fixed cost |
| **Slow listings** | `ls` or file discovery takes forever |
| **Driver OOM** | Spark driver runs out of memory tracking files |

### The Rule of Thumb

> **Target file size: 128MB - 1GB**
> **Maximum partitions: < 100,000**

### Solutions

**1. Choose low-cardinality partition keys:**
```python
# GOOD: Date, Region, Country
.partitionBy("date", "region")

# BAD: User ID, Session ID, Timestamp
.partitionBy("user_id")  # Millions of partitions!
```

**2. Use Delta Lake Auto-Optimize:**
```sql
ALTER TABLE gaming_sessions 
SET TBLPROPERTIES (
    delta.autoOptimize.optimizeWrite = true,
    delta.autoOptimize.autoCompact = true
);
```

**3. Run OPTIMIZE command:**
```sql
-- Compact small files into larger ones
OPTIMIZE gaming_sessions
WHERE year = 2024 AND month = 1;

-- Also apply Z-ORDER for better clustering
OPTIMIZE gaming_sessions
ZORDER BY (player_id);
```

---

## 🪣 Bucketing

### Concept

Data is distributed into a **fixed number of files (buckets)** based on hash of bucket column.

```
/data/gaming_sessions/
├── part-00000.parquet  (players where hash(player_id) % 256 = 0)
├── part-00001.parquet  (players where hash(player_id) % 256 = 1)
├── part-00002.parquet  (players where hash(player_id) % 256 = 2)
...
├── part-00255.parquet  (players where hash(player_id) % 256 = 255)
```

### When Bucketing Helps

**Scenario:** You frequently join `fact_sessions` with `dim_player` on `player_id`

Without bucketing:
```
SHUFFLE! All data moves across network
Nodes exchange data to co-locate matching player_ids
```

With bucketing (same bucket count on both tables):
```
NO SHUFFLE needed!
Bucket 0 from fact joins with bucket 0 from dim
Data is already co-located
```

### Creating Bucketed Tables

```sql
CREATE TABLE fact_sessions (
    session_id BIGINT,
    player_id STRING,
    duration_sec INT
)
USING DELTA
CLUSTERED BY (player_id) INTO 256 BUCKETS;

CREATE TABLE dim_player (
    player_id STRING,
    gamertag STRING,
    country STRING
)
USING DELTA
CLUSTERED BY (player_id) INTO 256 BUCKETS;

-- Now joins are shuffle-free!
SELECT f.*, p.gamertag
FROM fact_sessions f
JOIN dim_player p ON f.player_id = p.player_id;
```

---

## ⚖️ Partitioning vs Bucketing Decision Matrix

```
┌───────────────────────────────────────────────────────────────┐
│                 What's your primary use case?                  │
└─────────────────────────────┬─────────────────────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          │ Filter queries              Join queries
          │ (WHERE date = ?)           (ON user_id = ?)
          ▼                                      ▼
  ┌───────────────┐                      ┌───────────────┐
  │ PARTITIONING  │                      │   BUCKETING   │
  │               │                      │               │
  │ Low cardinality│                      │High cardinality│
  │ date, region  │                      │ user_id, device│
  └───────────────┘                      └───────────────┘
```

### Use Partitioning When:
- Queries filter on specific column values (`WHERE date = '2024-01-15'`)
- Column has **low cardinality** (< 10,000 distinct values)
- Time-based data with date queries
- Need to easily add/remove data (just add/delete directories)

### Use Bucketing When:
- Frequent **joins** on high-cardinality columns
- Column has **high cardinality** (millions of values)
- Can accept fixed bucket count (hard to change later)
- Both tables in join use same bucket count

### Use Both Together:
```sql
CREATE TABLE gaming_sessions
USING DELTA
PARTITIONED BY (date)           -- For time-based filtering
CLUSTERED BY (player_id) INTO 256 BUCKETS;  -- For join optimization
```

---

## 🔥 ADLS Gen2 Partitioning Best Practices

### Hierarchical Namespace Advantage

ADLS Gen2 with hierarchical namespace makes partition operations **atomic**:

```python
# This is atomic with hierarchical namespace
df.write \
    .partitionBy("date") \
    .format("delta") \
    .mode("overwrite") \
    .option("replaceWhere", "date = '2024-01-15'") \
    .save("abfss://container@storage.dfs.core.windows.net/gaming_sessions")
```

### Recommended Partition Strategy for ADLS

| Data Type | Partition Strategy | Reasoning |
|-----------|-------------------|-----------|
| **Event data** | `year/month/day` | Natural time-based queries |
| **Regional data** | `region/date` | Filter by region first, then time |
| **Multi-tenant** | `tenant_id/date` | Isolate tenant data |
| **Streaming** | `date/hour` | Hourly granularity for near-real-time |

---

## 💡 Z-Ordering (Delta Lake Specific)

### What Is It?

Z-Ordering **co-locates related data** within files for faster reads. Unlike partitioning (directories) or bucketing (fixed files), Z-ordering optimizes **within** files.

```sql
OPTIMIZE gaming_sessions
ZORDER BY (player_id, game_id);
```

### How It Works

```
Before Z-ORDER:
File 1: players A, Z, C, M, K...
File 2: players B, X, D, L, J...

After Z-ORDER by player_id:
File 1: players A, B, C, D, E...
File 2: players F, G, H, I, J...

Query for player 'B' now reads only File 1!
```

### When to Use Z-Ordering

- Columns you frequently filter on but can't partition by (high cardinality)
- Multiple columns in WHERE clauses
- Use **up to 4 columns** (effectiveness decreases with more)

---

## 🎯 Interview Answer Framework

When asked about partitioning:

### 1. Ask About Query Patterns

> *"What are the typical query patterns? Are we filtering by date? Joining on user_id?"*

### 2. Propose Partition Strategy

> *"I'd partition by date for two reasons:*
> - *Most queries filter on date ranges*
> - *Date has low cardinality (365 values per year)*
> - *Easy to manage TTL by dropping old partitions"*

### 3. Address Small Files

> *"To avoid the small files problem:*
> - *Enable Delta autoOptimize*
> - *Run OPTIMIZE weekly*
> - *Limit partition granularity (day, not hour)"*

### 4. Consider Z-Ordering

> *"For frequently filtered columns like player_id, I'd use Z-ORDER instead of partitioning to avoid too many partitions."*

---

## 🔥 Practice Question

**Scenario:** *"You have IoT sensor data: 10 million events per day from 100,000 devices across 5 regions. Design the partition strategy."*

**Consider:**
- Daily data size: ~10GB
- Queries: Regional dashboards, device-specific lookups
- Retention: 2 years

**Challenge:** If you partition by `device_id`, you get 100K partitions per day!

---

## 📖 Next Topic

Continue to [Cosmos DB Partition Key](./04-cosmos-db-partition-key.md) to learn NoSQL partitioning strategy.
