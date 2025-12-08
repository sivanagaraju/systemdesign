# Skew Handling & Salting

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Top Spark Question)

## The Core Question

*"Your Spark job is running for hours while most executors are idle but one is at 100%. What's happening?"*

This is **data skew** - the #1 performance killer in distributed data processing.

---

## 🤔 What Is Data Skew?

Data skew occurs when data is **unevenly distributed** across partitions, causing some tasks to process much more data than others.

```
SKEWED DISTRIBUTION (Bad)
┌────────────────────────────────────────────────────────────┐
│ Partition 1: ██                          (1% of data)      │
│ Partition 2: ███                         (2% of data)      │
│ Partition 3: ███████████████████████████████ (70% of data) │ ← Straggler!
│ Partition 4: █████                       (5% of data)      │
│ Partition 5: ██████████████████████      (22% of data)     │
└────────────────────────────────────────────────────────────┘

Job completes when the SLOWEST task finishes → 70% task dominates runtime!
```

### Common Causes

| Cause | Example |
|-------|---------|
| **Popular keys** | 50% of orders from "Amazon" in retailer analysis |
| **Null values** | Millions of rows with `customer_id = NULL` |
| **Date skew** | Black Friday has 10x normal traffic |
| **Default values** | All errors have `error_code = -1` |

---

## 🔍 How to Detect Skew

### Method 1: Spark UI

Look at the **Summary Metrics** in the Stages tab:

```
Task Duration Summary:
  Min: 2 seconds
  25th percentile: 5 seconds
  Median: 8 seconds
  75th percentile: 15 seconds
  Max: 45 MINUTES  ← This is the straggler!
```

### Method 2: Key Distribution Analysis

```python
# Check key distribution before join/groupBy
df.groupBy("customer_id") \
  .count() \
  .orderBy(desc("count")) \
  .show(10)

# Output shows skew:
# +-------------+--------+
# |customer_id  |count   |
# +-------------+--------+
# |AMAZON       |5000000 |  ← 50x larger than others!
# |WALMART      |100000  |
# |TARGET       |95000   |
# +-------------+--------+
```

### Method 3: Partition Size Check

```python
from pyspark.sql.functions import spark_partition_id

# Check data per partition
df.withColumn("partition_id", spark_partition_id()) \
  .groupBy("partition_id") \
  .count() \
  .orderBy("partition_id") \
  .show()
```

---

## 🧂 The Salting Technique

### Concept

**Salting** adds a random suffix to skewed keys, spreading them across multiple partitions.

```
Before Salting:
┌──────────────────────────────────────────────────────┐
│ Key: "AMAZON"    → Partition 3 (via hash("AMAZON"))  │
│ Key: "AMAZON"    → Partition 3                       │
│ Key: "AMAZON"    → Partition 3  (all 5M rows here!)  │
│ Key: "WALMART"   → Partition 7                       │
└──────────────────────────────────────────────────────┘

After Salting:
┌──────────────────────────────────────────────────────┐
│ Key: "AMAZON_0"  → Partition 2                       │
│ Key: "AMAZON_1"  → Partition 5                       │
│ Key: "AMAZON_2"  → Partition 8  (distributed!)       │
│ Key: "AMAZON_3"  → Partition 1                       │
│ Key: "WALMART"   → Partition 7                       │
└──────────────────────────────────────────────────────┘
```

### Implementation: Salted Join

```python
from pyspark.sql.functions import col, concat, lit, floor, rand, explode, array

# Parameters
SALT_BUCKETS = 10  # Number of salt values

# ============================================
# STEP 1: Salt the LARGE (skewed) table
# ============================================
large_df_salted = large_df.withColumn(
    "salted_key",
    concat(
        col("customer_id"),
        lit("_"),
        floor(rand() * SALT_BUCKETS).cast("string")
    )
)

# ============================================
# STEP 2: Explode the SMALL table
# ============================================
# Create array of salt values [0, 1, 2, ..., 9]
salt_values = [str(i) for i in range(SALT_BUCKETS)]

small_df_exploded = small_df.withColumn(
    "salt",
    explode(array([lit(s) for s in salt_values]))
).withColumn(
    "salted_key",
    concat(col("customer_id"), lit("_"), col("salt"))
)

# ============================================
# STEP 3: Join on salted keys
# ============================================
result = large_df_salted.join(
    small_df_exploded,
    on="salted_key",
    how="inner"
).drop("salted_key", "salt")
```

### Visualization of Salted Join

```
Large Table (5M AMAZON rows):                Small Table (reference):
┌─────────────────────────┐                  ┌─────────────────────┐
│ customer_id │ amount    │                  │ customer_id │ region│
├─────────────┼───────────┤                  ├─────────────┼───────┤
│ AMAZON      │ 100       │                  │ AMAZON      │ US    │
│ AMAZON      │ 200       │                  │ WALMART     │ US    │
│ AMAZON      │ 150       │                  └─────────────────────┘
│ ...         │ ...       │                         │
└─────────────────────────┘                         │ EXPLODE with salts
           │                                        ▼
           │ ADD random salt              ┌─────────────────────────┐
           ▼                              │ salted_key    │ region  │
┌───────────────────────────┐             ├───────────────┼─────────┤
│ salted_key    │ amount    │             │ AMAZON_0      │ US      │
├───────────────┼───────────┤             │ AMAZON_1      │ US      │
│ AMAZON_3      │ 100       │             │ AMAZON_2      │ US      │
│ AMAZON_7      │ 200       │             │ ...           │ ...     │
│ AMAZON_1      │ 150       │             │ AMAZON_9      │ US      │
│ ...           │ ...       │             │ WALMART_0     │ US      │
└───────────────────────────┘             │ ...           │ ...     │
           │                              └─────────────────────────┘
           │                                        │
           └──────────────┬─────────────────────────┘
                          │ JOIN on salted_key
                          ▼
                   EVENLY DISTRIBUTED!
```

---

## 🔧 Salting for GroupBy/Aggregations

For aggregations, use a **two-phase approach**:

```python
from pyspark.sql.functions import sum as spark_sum

# Phase 1: Partial aggregation with salt
partial_agg = df.withColumn(
    "salt",
    floor(rand() * SALT_BUCKETS)
).groupBy("customer_id", "salt") \
 .agg(spark_sum("amount").alias("partial_sum"))

# Phase 2: Final aggregation (removes salt)
final_agg = partial_agg.groupBy("customer_id") \
    .agg(spark_sum("partial_sum").alias("total_amount"))
```

---

## ⚡ Adaptive Query Execution (AQE)

### Spark 3.0+ Built-in Skew Handling

Spark 3.0 introduced **Adaptive Query Execution** which can automatically detect and handle skew!

```python
# Enable AQE (often on by default in Databricks)
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

# Configure thresholds
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")  
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```

### How AQE Skew Handling Works

```
Without AQE:
┌──────────────────────────────────────────────┐
│ Task 1: █████████████████████████ (skewed)   │ 45 min
│ Task 2: ██                                   │ 2 min
│ Task 3: ███                                  │ 3 min
└──────────────────────────────────────────────┘
Total time: 45 minutes (waiting for Task 1)

With AQE Skew Handling:
┌──────────────────────────────────────────────┐
│ Task 1a: ████████                            │ 10 min
│ Task 1b: ████████ (auto-split!)              │ 10 min
│ Task 1c: █████████                           │ 12 min
│ Task 2:  ██                                  │ 2 min
│ Task 3:  ███                                 │ 3 min
└──────────────────────────────────────────────┘
Total time: 12 minutes!
```

### When AQE Isn't Enough

AQE handles **join skew** well, but you might still need manual salting for:
- Aggregation skew (`groupBy` on skewed keys)
- Very extreme skew (single key is 90% of data)
- Custom business logic requirements

---

## 🎯 Interview Answer Framework

### Step 1: Identify the Problem

> *"First, I'd check the Spark UI for straggler tasks. If task duration has high variance (e.g., median 5 seconds, max 45 minutes), that indicates skew."*

### Step 2: Analyze the Root Cause

> *"I'd analyze key distribution to find the hot keys:*
> ```python
> df.groupBy("key").count().orderBy(desc("count")).show(10)
> ```
> *This reveals if one key dominates the data."*

### Step 3: Choose Solution

> *"For Spark 3.0+, I'd first enable AQE skew handling. If that's insufficient, I'd use manual salting."*

### Step 4: Explain Salting

> *"Salting adds a random suffix (0-9) to hot keys, spreading them across partitions. The small table is exploded to match all salt values."*

### Step 5: Trade-offs

> *"The trade-off is:*
> - *Small table grows 10x (one row per salt)*
> - *Extra shuffle for the salt join*
> - *But execution time drops from hours to minutes*"

---

## ⚠️ Common Interview Traps

### Trap 1: "Just increase partitions"

**Response:** More partitions doesn't help. If one key has 5M rows and goes to one partition, adding more partitions won't split that key.

### Trap 2: "Use broadcast join"

**Response:** Broadcast works for small tables, but doesn't solve skew when joining two large tables.

### Trap 3: "Filter out the hot key"

**Response:** Sometimes valid (e.g., remove nulls), but usually the hot key contains important business data.

---

## 💡 Quick Reference: Skew Solutions

| Scenario | Solution |
|----------|----------|
| Join skew, Spark 3.0+ | Enable AQE skew handling |
| Join skew, Spark 2.x | Manual salting |
| GroupBy skew | Two-phase aggregation with salt |
| Null key skew | Filter nulls first, process separately |
| Known hot keys | Broadcast just the hot keys |

---

## 📖 Next Topic

Continue to [Broadcast vs Shuffle Joins](./02-broadcast-vs-shuffle-joins.md) to understand join strategies.
