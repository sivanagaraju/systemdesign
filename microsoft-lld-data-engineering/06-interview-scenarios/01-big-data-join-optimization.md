# Scenario: Big Data Join Optimization

## 📝 Problem Statement

> *"We have a 10TB table and a 50MB table. How does Spark join them, and how would you optimize it?"*

**Context:** Orders fact table (10TB) needs to join with Product dimension (50MB).

---

## ⏱️ Time: 3 minutes to explain

---

## 🎯 Model Answer

### 1. Clarify (30 seconds)

*"Before I start, let me clarify:*
- *Is the 50MB table relatively static, or does it change frequently?*
- *What's the join key - product_id?*
- *Are we optimizing for latency or throughput?"*

### 2. Identify Pattern (30 seconds)

*"This is a classic large-small table join. The 50MB table easily fits in executor memory, making it ideal for a **Broadcast Hash Join**."*

### 3. Explain Mechanism (1 minute)

*"Here's how Broadcast Join works:*

1. *Driver collects the 50MB table*
2. *Serializes and broadcasts to every executor*
3. *Each executor has a local copy*
4. *Each executor joins its 10TB partition locally*
5. *No shuffle of the 10TB table occurs!"*

```
Driver: Serialize 50MB table
           │
    ┌──────┴──────┬──────────────┐
    ▼             ▼              ▼
Executor 1    Executor 2    Executor 3
[50MB copy]   [50MB copy]   [50MB copy]
[10TB part]   [10TB part]   [10TB part]
    │             │              │
    ▼             ▼              ▼
Local JOIN    Local JOIN    Local JOIN
```

### 4. Implementation (1 minute)

```python
from pyspark.sql.functions import broadcast

# Force broadcast (explicit)
result = orders_10tb.join(
    broadcast(products_50mb),
    on="product_id",
    how="inner"
)

# Or configure threshold
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100MB")

# Verify with explain
result.explain(True)
# Should show: BroadcastHashJoin
```

### 5. Trade-offs (30 seconds)

*"Trade-offs to mention:*
- *Driver memory must hold 50MB table*
- *Network cost to broadcast to all executors*
- *If product table grows to 500MB, might need to increase threshold*
- *If table updates hourly, broadcast cache needs refresh"*

---

## ❌ Common Mistakes

1. **Not mentioning broadcast** - Shuffle join is wrong answer
2. **Forgetting driver memory** - Broadcast requires driver to hold table
3. **No verification** - Always mention `explain()` to confirm plan

---

## 📖 Next Scenario

Continue to [Data Quality Validator](./02-data-quality-validator.md).
