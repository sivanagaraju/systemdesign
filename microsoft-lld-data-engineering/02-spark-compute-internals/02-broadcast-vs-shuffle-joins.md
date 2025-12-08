# Broadcast vs Shuffle Joins

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Top Microsoft Question)

## The Core Question

*"We have a 10TB table and a 50MB table. How does Spark join them, and how would you optimize it?"*

This is the **#1 join optimization question** at Microsoft.

---

## 🎯 Quick Answer

> *"Use a **Broadcast Join**. The 50MB table is serialized and sent to every worker node, then joined locally with partitions of the 10TB table. This avoids shuffling 10TB across the network."*

---

## 📊 Spark Join Strategies Overview

| Join Strategy | When Used | Shuffle Required? | Best For |
|--------------|-----------|-------------------|----------|
| **Broadcast Hash** | One side < 10MB (default) | ❌ No | Large + Small table |
| **Shuffle Hash** | Medium tables, no sort | ✅ Yes (both sides) | Equal-sized unsorted |
| **Sort Merge** | Large + Large tables | ✅ Yes (both sides) | Large sorted tables |
| **Broadcast Nested Loop** | Cross joins, non-equi | ❌ No | Small cross joins |

---

## 📡 Broadcast Hash Join (BHJ)

### How It Works

```
DRIVER collects small table
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  DRIVER: Serialized small_table (50MB)                          │
│           │                                                     │
│     ┌─────┴─────┬───────────────┬───────────────┐               │
│     ▼           ▼               ▼               ▼               │
│  ┌──────┐   ┌──────┐       ┌──────┐       ┌──────┐              │
│  │Worker│   │Worker│       │Worker│       │Worker│              │
│  │  1   │   │  2   │       │  3   │       │  4   │              │
│  ├──────┤   ├──────┤       ├──────┤       ├──────┤              │
│  │small │   │small │       │small │       │small │              │
│  │table │   │table │       │table │       │table │              │
│  │(copy)│   │(copy)│       │(copy)│       │(copy)│              │
│  ├──────┤   ├──────┤       ├──────┤       ├──────┤              │
│  │large │   │large │       │large │       │large │              │
│  │part 1│   │part 2│       │part 3│       │part 4│              │
│  └──┬───┘   └──┬───┘       └──┬───┘       └──┬───┘              │
│     │          │               │               │                │
│     ▼          ▼               ▼               ▼                │
│   JOIN       JOIN            JOIN            JOIN               │
│  locally    locally         locally         locally             │
└─────────────────────────────────────────────────────────────────┘

NO SHUFFLE of large table! Each partition joins with its local copy.
```

### When Spark Uses Broadcast Automatically

```python
# Default broadcast threshold: 10MB
spark.conf.get("spark.sql.autoBroadcastJoinThreshold")  # "10485760" (10MB)
```

If one table is under 10MB, Spark automatically broadcasts it.

### Force Broadcast with Hint

```python
from pyspark.sql.functions import broadcast

# Method 1: broadcast() function
result = large_df.join(
    broadcast(small_df),
    on="customer_id",
    how="inner"
)

# Method 2: SQL hint
spark.sql("""
    SELECT /*+ BROADCAST(small_table) */ *
    FROM large_table
    JOIN small_table ON large_table.id = small_table.id
""")

# Method 3: DataFrame hint
result = large_df.join(
    small_df.hint("broadcast"),
    on="customer_id"
)
```

### Increasing Broadcast Threshold

```python
# For tables larger than 10MB but still "small enough"
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "500MB")  # 500MB

# Or disable auto-broadcast entirely
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")
```

---

## 🔀 Shuffle Hash Join

### How It Works

Both tables are **shuffled by join key** so matching keys land on the same partition.

```
Table A:                          Table B:
┌──────────────┐                  ┌──────────────┐
│ id │ value_a │                  │ id │ value_b │
├────┼─────────┤                  ├────┼─────────┤
│ 1  │ "a"     │                  │ 2  │ "x"     │
│ 2  │ "b"     │                  │ 3  │ "y"     │
│ 3  │ "c"     │                  │ 1  │ "z"     │
└────┴─────────┘                  └────┴─────────┘
        │                                │
        │     SHUFFLE by hash(id)        │
        ▼                                ▼
┌───────────────────────────────────────────────────┐
│ Partition 0:  A(id=1), A(id=3), B(id=1), B(id=3)  │
│ Partition 1:  A(id=2), B(id=2)                    │
└───────────────────────────────────────────────────┘
        │
        │ Local hash join within each partition
        ▼
┌──────────────────────────┐
│ id │ value_a │ value_b   │
├────┼─────────┼───────────┤
│ 1  │ "a"     │ "z"       │
│ 2  │ "b"     │ "x"       │
│ 3  │ "c"     │ "y"       │
└────┴─────────┴───────────┘
```

### When Used
- Neither table fits in memory for broadcast
- Data isn't pre-sorted
- `spark.sql.join.preferSortMergeJoin` is false

---

## 🔄 Sort Merge Join (SMJ)

### How It Works

1. Both tables **shuffled** by join key
2. Both sides **sorted** within partitions
3. **Merge** sorted streams (like merge sort)

```
Table A (shuffled, sorted):       Table B (shuffled, sorted):
┌──────────────┐                  ┌──────────────┐
│ id │ value_a │                  │ id │ value_b │
├────┼─────────┤                  ├────┼─────────┤
│ 1  │ "a"     │ ←──────────────► │ 1  │ "z"     │  MATCH!
│ 2  │ "b"     │ ←────────────────┤ 2  │ "x"     │  MATCH!
│ 3  │ "c"     │ ←────────────────┤ 3  │ "y"     │  MATCH!
│ 4  │ "d"     │  (no match)      │              │
└────┴─────────┘                  └────┴─────────┘

Both pointers advance in order - very efficient!
```

### When Sort Merge Wins
- Very large tables (can't broadcast either)
- Already sorted by join key (skips sort phase)
- Default for large-large joins in Spark

### Force Sort Merge Join

```python
spark.conf.set("spark.sql.join.preferSortMergeJoin", "true")  # Default

# Or via SQL hint
spark.sql("""
    SELECT /*+ MERGE(table_a, table_b) */ *
    FROM table_a
    JOIN table_b ON table_a.id = table_b.id
""")
```

---

## ⚖️ Comparison Table

| Aspect | Broadcast Hash | Shuffle Hash | Sort Merge |
|--------|----------------|--------------|------------|
| **Shuffle** | None for large table | Both tables | Both tables |
| **Memory** | Small table in memory | Hash table in memory | Sorted buffer |
| **Sorting** | None | None | Required |
| **Best For** | Large + Small | Medium + Medium | Large + Large |
| **Scalability** | Limited by driver | Limited by partition | Best for huge data |

---

## 🎯 The 10TB + 50MB Interview Answer

### Full Answer Framework

> **Step 1: Identify the pattern**
> *"This is a classic large table + small lookup table join. The small table (50MB) fits easily in memory."*

> **Step 2: Recommend broadcast**
> *"I'd use a Broadcast Hash Join. Spark will:*
> 1. *Collect the 50MB table to the driver*
> 2. *Serialize it and broadcast to all executors*
> 3. *Each executor joins its local 10TB partitions with the broadcast copy*
> 4. *No shuffle of the 10TB table occurs"*

> **Step 3: Implementation**
> ```python
> from pyspark.sql.functions import broadcast
> 
> result = large_10tb_df.join(
>     broadcast(small_50mb_df),
>     on="join_key",
>     how="inner"
> )
> ```

> **Step 4: Verify with explain**
> ```python
> result.explain(True)
> # Should show: BroadcastHashJoin
> ```

> **Step 5: Mention limits**
> *"Default threshold is 10MB. For 50MB, I'd increase it:*
> ```python
> spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100MB")
> ```
> *"*

---

## 💡 Reading Explain Plans

```python
df.explain(True)  # Shows all plan stages
```

**Broadcast Join in explain output:**
```
== Physical Plan ==
*(5) BroadcastHashJoin [id#1], [id#2], Inner, BuildRight, false
:- *(5) Project [id#1, value#2]
:  +- *(5) Filter isnotnull(id#1)
:     +- FileScan parquet [id#1,value#2]
+- BroadcastExchange HashedRelationBroadcastMode(List(id#2))  ← BROADCAST!
   +- *(1) Project [id#2, lookup#3]
      +- FileScan parquet [id#2,lookup#3]
```

**Shuffle Join (Sort Merge) in explain output:**
```
== Physical Plan ==
*(5) SortMergeJoin [id#1], [id#2], Inner
:- *(2) Sort [id#1 ASC NULLS FIRST], false, 0
:  +- Exchange hashpartitioning(id#1, 200)  ← SHUFFLE!
:     +- *(1) Project [id#1, value#2]
:        +- FileScan parquet
+- *(4) Sort [id#2 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(id#2, 200)  ← SHUFFLE!
      +- *(3) Project [id#2, lookup#3]
         +- FileScan parquet
```

---

## ⚠️ Common Interview Traps

### Trap 1: "What if the small table is 500MB?"

**Response:** 
> *"500MB is still broadcastable. I'd increase the threshold:*
> ```python
> spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "1GB")
> ```
> *But I'd verify executor memory can handle it. Each executor gets a copy, so 10 executors = 5GB total memory for broadcasts."*

### Trap 2: "What if both tables are 10TB?"

**Response:**
> *"Then broadcast isn't an option. I'd use Sort Merge Join. But I'd check:*
> - *Can I pre-partition both tables by join key? (Bucketing)*
> - *Is there data skew on the join key? (Salting)*
> - *Can I filter either table first to reduce size?"*

### Trap 3: "Why not always broadcast?"

**Response:**
> *"Broadcast has limits:*
> - *Driver memory must hold the table*
> - *Network transfer to every executor*
> - *Very large broadcasts can cause OOM*
> - *If table changes, must re-broadcast each job"*

---

## 🔥 Practice Question

**Scenario:** *"You're joining a 5TB fact table with a 100MB dimension table, but the dimension table changes hourly. How do you handle this?"*

**Consider:**
- Broadcast is still appropriate (100MB is small)
- But how do you handle hourly updates?
- Delta Lake or external table refresh?

---

## 📖 Next Topic

Continue to [Catalyst Optimizer](./03-catalyst-optimizer.md) to understand how Spark optimizes query plans.
