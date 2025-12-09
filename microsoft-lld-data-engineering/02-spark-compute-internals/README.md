# 02 - Spark & Compute Internals

> **Core LLD Skill:** Understanding Spark execution model for performance optimization

Microsoft relies heavily on **Apache Spark** (via Azure Databricks & Synapse). They won't just ask "how to code it" - they'll ask "how it works under the hood."

---

## 📚 Topics in This Section

| File | Topic | Key Concepts |
|------|-------|--------------|
| [01-skew-handling-salting.md](./01-skew-handling-salting.md) | Data Skew | Salting, AQE skew handling |
| [02-broadcast-vs-shuffle-joins.md](./02-broadcast-vs-shuffle-joins.md) | Join Strategies | Broadcast, Sort-Merge, Shuffle Hash |
| [03-catalyst-optimizer.md](./03-catalyst-optimizer.md) | Query Optimization | Logical/Physical plans, explain() |
| [04-file-formats-deep-dive.md](./04-file-formats-deep-dive.md) | Storage Formats | Parquet, Avro, ORC, Delta |
| [05-debugging-oom-errors.md](./05-debugging-oom-errors.md) | Troubleshooting | OOM analysis, memory tuning |
| [06-advanced-performance-optimization.md](./06-advanced-performance-optimization.md) | **Performance Master Guide** | SPAMS framework, all optimization techniques |

---

## 🎯 Common Interview Questions

1. *"Your Spark job is failing with an OOM error. How do you debug it?"*
2. *"We have a 10TB table and a 50MB table. How does Spark join them?"*
3. *"What is data skew and how do you fix it?"*
4. *"Explain how Spark optimizes your SQL query"*

---

## 🏗️ Spark Execution Model Overview

```
┌───────────────────────────────────────────────────────────────┐
│                      DRIVER                                    │
│ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│ │ Logical     │─►│ Catalyst    │─►│ Physical    │              │
│ │ Plan        │  │ Optimizer   │  │ Plan        │              │
│ └─────────────┘  └─────────────┘  └─────────────┘              │
│                                          │                     │
│                    ┌─────────────────────┴──────────────┐      │
│                    │            DAG Scheduler           │      │
│                    └─────────────────────┬──────────────┘      │
└──────────────────────────────────────────┼─────────────────────┘
                                           │
           ┌───────────────────────────────┼───────────────────────────┐
           │                               ▼                           │
           │ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
           │ │  Executor 1 │  │  Executor 2 │  │  Executor 3 │         │
           │ │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │         │
           │ │ │ Task 1  │ │  │ │ Task 4  │ │  │ │ Task 7  │ │         │
           │ │ │ Task 2  │ │  │ │ Task 5  │ │  │ │ Task 8  │ │         │
           │ │ │ Task 3  │ │  │ │ Task 6  │ │  │ │ Task 9  │ │         │
           │ │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │         │
           │ └─────────────┘  └─────────────┘  └─────────────┘         │
           │                        CLUSTER                             │
           └────────────────────────────────────────────────────────────┘
```

---

## 🔑 Key Concepts Quick Reference

### Shuffle Operations
Operations that redistribute data across the cluster:
- `groupBy()`, `reduceByKey()`
- `join()` (except broadcast)
- `distinct()`, `repartition()`

**Why it matters:** Shuffles are expensive (network I/O). Minimize them!

### Stages & Tasks
- **Job** = Triggered by action (collect, save, count)
- **Stage** = Sequence of transformations without shuffle
- **Task** = Work unit on a single partition

### Memory Areas
```
┌─────────────────────────────────────────┐
│              Executor Memory            │
│  ┌──────────────────────────────────┐   │
│  │     Execution Memory (60%)       │   │  ← Shuffles, joins, sorts
│  │         (Unified Pool)           │   │
│  ├──────────────────────────────────┤   │
│  │     Storage Memory (40%)         │   │  ← Cached RDDs, broadcasts
│  │         (Unified Pool)           │   │
│  ├──────────────────────────────────┤   │
│  │     User Memory                  │   │  ← Your code's objects
│  ├──────────────────────────────────┤   │
│  │     Reserved (300MB)             │   │  ← System overhead
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

## 💡 Performance Optimization Checklist

| Category | Optimization | How |
|----------|--------------|-----|
| **Joins** | Use broadcast for small tables | `.hint("broadcast")` |
| **Skew** | Salt skewed keys | Add random suffix |
| **Files** | Target 128MB-1GB files | OPTIMIZE, coalesce |
| **Partitions** | 2-3x cores for parallelism | `spark.sql.shuffle.partitions` |
| **Caching** | Cache frequently used DataFrames | `.cache()` or `.persist()` |
| **Serialization** | Use Kryo serializer | `spark.serializer` config |

---

## 📖 Start Here

Begin with [Skew Handling & Salting](./01-skew-handling-salting.md) to understand the most common performance killer.
