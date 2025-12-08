# Debugging OOM Errors

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Common Scenario Question)

## The Core Question

*"Your Spark job is failing with an OutOfMemory error. How do you debug and fix it?"*

This tests your **troubleshooting methodology** and memory model understanding.

---

## 🔴 Types of OOM Errors

| Error Location | Error Message | Cause |
|----------------|---------------|-------|
| **Driver OOM** | `java.lang.OutOfMemoryError` on driver | Collecting too much data to driver |
| **Executor OOM** | `Container killed by YARN for exceeding memory limits` | Task processing too much data |
| **Shuffle OOM** | `OutOfMemoryError: Java heap space` during shuffle | Shuffle spill to disk failed |

---

## 🛠️ Debugging Workflow

```
OOM Error Detected
        │
        ▼
┌───────────────────────────────┐
│ 1. Identify WHERE it failed   │
│    Driver or Executor?        │
└───────────────┬───────────────┘
                │
        ┌───────┴───────┐
        ▼               ▼
    DRIVER          EXECUTOR
        │               │
        ▼               ▼
┌───────────────┐ ┌───────────────────────┐
│ Check for:    │ │ Check for:            │
│ - collect()   │ │ - Data skew           │
│ - toPandas()  │ │ - Large partitions    │
│ - broadcast   │ │ - Expensive operations│
│ - show(all)   │ │   (explode, UDFs)     │
└───────────────┘ └───────────────────────┘
```

---

## 🖥️ Driver OOM

### Common Causes

```python
# CAUSE 1: Collecting all data to driver
df.collect()  # Brings ALL rows to driver memory!

# CAUSE 2: Converting to Pandas
df.toPandas()  # Entire DataFrame to driver

# CAUSE 3: Broadcast too large table
broadcast(large_df)  # Driver must hold entire table

# CAUSE 4: Too many partitions metadata
df.repartition(1000000)  # Driver tracks all partition metadata
```

### Solutions

```python
# Instead of collect(), use:
df.take(100)           # Only first 100 rows
df.show(100)           # Display 100 rows
df.limit(100).collect() # Collect limited data

# Instead of toPandas() for large data:
df.write.csv("/output")  # Write to storage
# Then read with pandas in chunks

# For broadcast:
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100MB")
# Never broadcast tables > driver memory

# Increase driver memory (if necessary):
spark.conf.set("spark.driver.memory", "8g")
```

---

## ⚙️ Executor OOM

### Common Causes

| Cause | Symptoms |
|-------|----------|
| **Data skew** | One task takes much longer, then OOM |
| **Large partitions** | Task processing GB of data per partition |
| **Expensive UDFs** | Memory-heavy operations in Python UDFs |
| **Cartesian joins** | Exploding row count |
| **Wide transformations** | Many columns per row |

### Debugging Steps

**Step 1: Check Spark UI**

```
Stages tab → Click on failed stage → Task metrics

Look for:
- Max task duration >> median duration (skew)
- Shuffle read size per task (uneven distribution)
- GC time (high = memory pressure)
```

**Step 2: Check Partition Sizes**

```python
# Check how much data per partition
from pyspark.sql.functions import spark_partition_id

df.groupBy(spark_partition_id().alias("partition")) \
  .count() \
  .orderBy("count") \
  .show()

# If one partition has 10M rows while others have 100K → SKEW!
```

**Step 3: Analyze the Stage**

```python
# What operation failed?
df.explain(True)

# Look for:
# - HashAggregate (might cause memory issues with many groups)
# - SortMergeJoin (sorting large data)
# - Explode (row multiplication)
```

### Solutions

```python
# SOLUTION 1: Increase partitions (reduce data per partition)
spark.conf.set("spark.sql.shuffle.partitions", "500")  # Default 200
df.repartition(500)

# SOLUTION 2: Increase executor memory
spark.conf.set("spark.executor.memory", "16g")
spark.conf.set("spark.executor.memoryOverhead", "4g")  # For Python/off-heap

# SOLUTION 3: Fix data skew (see salting technique)
# Add random salt to skewed keys

# SOLUTION 4: Enable spilling to disk
spark.conf.set("spark.shuffle.spill", "true")  # Default
spark.conf.set("spark.memory.fraction", "0.6")

# SOLUTION 5: Use more efficient operations
# Instead of UDF:
df.withColumn("upper_name", upper(col("name")))  # Built-in function
# Instead of:
@udf
def upper_udf(x): return x.upper()
df.withColumn("upper_name", upper_udf(col("name")))  # Python UDF (slower, more memory)
```

---

## 📊 Memory Tuning Parameters

### Executor Memory Layout

```
Total Executor Memory (spark.executor.memory = 16GB)
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Reserved Memory (300MB)                                   │
│   ───────────────────────────                               │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │           Unified Memory (60%)                      │   │
│   │  ┌──────────────────────┬──────────────────────┐    │   │
│   │  │ Execution (shuffles, │ Storage (cache,      │    │   │
│   │  │ joins, aggregations) │ broadcast)           │    │   │
│   │  │                      │                      │    │   │
│   │  │ ◄─── Can borrow ───► │                      │    │   │
│   │  └──────────────────────┴──────────────────────┘    │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   User Memory (40% - reserved)                              │
│   - Your code's objects                                     │
│   - UDF data structures                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Memory Overhead (spark.executor.memoryOverhead)
┌─────────────────────────────────────────────────────────────┐
│  Off-heap memory for:                                       │
│  - Python processes (PySpark)                               │
│  - Native libraries                                         │
│  - Container overhead                                       │
└─────────────────────────────────────────────────────────────┘
```

### Key Configuration

```python
# Executor JVM heap
spark.conf.set("spark.executor.memory", "16g")

# Off-heap/overhead (important for PySpark!)
spark.conf.set("spark.executor.memoryOverhead", "4g")  # Or 10-15% of executor memory

# Memory fraction for execution + storage
spark.conf.set("spark.memory.fraction", "0.6")  # Default

# Portion of unified memory for storage
spark.conf.set("spark.memory.storageFraction", "0.5")  # Default or execution
```

---

## 🎯 Interview Answer Framework

When asked about OOM debugging:

> **Step 1: Identify location**
> *"First, I check WHERE the OOM occurred - driver or executor. The error message indicates this."*

> **Step 2: Driver OOM**
> *"If driver, I look for collect(), toPandas(), or large broadcasts. I'd replace collect() with write() or limit()+collect()."*

> **Step 3: Executor OOM**
> *"If executor, I check the Spark UI for:*
> - *Task duration variance (indicates skew)*
> - *Shuffle read sizes (uneven = skew)*
> - *GC time (high = memory pressure)"*

> **Step 4: Fix based on root cause**
> *"Solutions depend on cause:*
> - *Skew: Use salting technique*
> - *Large partitions: Increase partition count*
> - *Memory: Increase executor.memory and memoryOverhead"*

---

## 🔥 Quick Diagnosis Commands

```python
# Check current memory settings
print(spark.conf.get("spark.driver.memory"))
print(spark.conf.get("spark.executor.memory"))
print(spark.conf.get("spark.sql.shuffle.partitions"))

# Check DataFrame partition count
print(f"Partitions: {df.rdd.getNumPartitions()}")

# Check data size estimate
df.cache()
df.count()  # Triggers caching
spark.catalog.clearCache()  # Check Storage tab in UI first

# Analyze skew
df.groupBy("key").count().orderBy(desc("count")).show(20)
```

---

## 📖 Next Section

Move to [03 - Pipeline Resilience](../03-pipeline-resilience/README.md) to learn about fault-tolerant pipeline design.
