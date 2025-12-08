# Catalyst Optimizer

> **Interview Frequency:** ⭐⭐⭐⭐ (Bonus Points Topic)

## The Core Question

*"How does Spark optimize your SQL query before executing it?"*

Understanding Catalyst shows **deep Spark knowledge** - it impresses interviewers.

---

## 🤔 What Is Catalyst?

Catalyst is Spark's **query optimizer**. It transforms your logical query into an optimized physical execution plan.

```
Your SQL Query
      │
      ▼
┌─────────────────┐
│ Parsed Logical  │  ← Just syntax, no optimization
│     Plan        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│Analyzed Logical │  ← Resolved table/column names
│     Plan        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│Optimized Logical│  ← Predicate pushdown, constant folding
│     Plan        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Physical Plan  │  ← Actual execution strategy (broadcast, shuffle)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Code Generation │  ← Java bytecode (Tungsten)
└─────────────────┘
```

---

## 🔍 Viewing Query Plans

```python
# Method 1: Simple physical plan
df.explain()

# Method 2: All plan stages
df.explain(True)  # or explain("extended")

# Method 3: Full details including generated code
df.explain("codegen")

# Method 4: Cost-based info (if statistics available)
df.explain("cost")

# Method 5: Formatted tree view (Spark 3.0+)
df.explain("formatted")
```

### Sample Output

```python
spark.sql("""
    SELECT customer_id, SUM(amount)
    FROM orders
    WHERE order_date = '2024-01-15'
    GROUP BY customer_id
""").explain(True)
```

```
== Parsed Logical Plan ==
'Aggregate ['customer_id], ['customer_id, 'SUM('amount)]
+- 'Filter ('order_date = '2024-01-15')
   +- 'UnresolvedRelation [orders]

== Analyzed Logical Plan ==
customer_id: string, sum(amount): decimal(38,2)
Aggregate [customer_id#1], [customer_id#1, sum(amount#3) AS sum(amount)#12]
+- Filter (order_date#2 = 2024-01-15)
   +- SubqueryAlias orders
      +- Relation[customer_id#1,order_date#2,amount#3] parquet

== Optimized Logical Plan ==
Aggregate [customer_id#1], [customer_id#1, sum(amount#3) AS sum(amount)#12]
+- Project [customer_id#1, amount#3]          ← Only needed columns!
   +- Filter (order_date#2 = 2024-01-15)
      +- Relation[customer_id#1,order_date#2,amount#3] parquet

== Physical Plan ==
*(2) HashAggregate(keys=[customer_id#1], functions=[sum(amount#3)])
+- Exchange hashpartitioning(customer_id#1, 200)  ← Shuffle for groupBy
   +- *(1) HashAggregate(keys=[customer_id#1], functions=[partial_sum(amount#3)])
      +- *(1) Project [customer_id#1, amount#3]
         +- *(1) Filter (order_date#2 = 2024-01-15)     ← Pushed to scan
            +- *(1) FileScan parquet [customer_id#1,order_date#2,amount#3]
                   PushedFilters: [IsNotNull(order_date), EqualTo(order_date,2024-01-15)]
```

---

## ⚡ Key Optimizations

### 1. Predicate Pushdown

Filters pushed down to the data source to skip reading unnecessary data.

```python
# Your query:
df.filter(df.date == "2024-01-15").select("customer_id", "amount")

# Without pushdown: Read all data, then filter
# With pushdown:    Skip reading non-matching partitions/row groups
```

**In explain output:**
```
FileScan parquet
  PushedFilters: [EqualTo(date,2024-01-15)]  ← Filter pushed to storage!
```

### 2. Projection Pruning

Only read columns that are actually used.

```python
# Your query:
df.select("customer_id", "amount")  # Only 2 of 50 columns

# Spark only reads these 2 columns from Parquet (columnar format advantage)
```

**In explain output:**
```
FileScan parquet [customer_id#1,amount#3]  ← Only 2 columns scanned
  Required: [customer_id,amount]
```

### 3. Constant Folding

Evaluates constant expressions at compile time.

```python
# Your query:
df.filter(df.amount > 100 + 50)

# Optimized to:
df.filter(df.amount > 150)  # Computed at planning time
```

### 4. Combine Filters

Multiple filters merged into one.

```python
# Your query:
df.filter(df.a > 10).filter(df.b < 20)

# Optimized to:
df.filter((df.a > 10) & (df.b < 20))  # Single filter operation
```

### 5. Boolean Expression Simplification

```python
# Your query:
df.filter((df.x > 5) & True)

# Optimized to:
df.filter(df.x > 5)  # Removed redundant True
```

---

## 🔧 Join Reordering

Catalyst reorders joins to minimize shuffling when statistics are available.

```python
# Your query: A JOIN B JOIN C

# If table sizes are: A=10TB, B=100MB, C=500MB
# Catalyst may reorder to: A JOIN (B JOIN C)
# So small tables join first, then join with large table
```

### Enable Cost-Based Optimization

```python
# Collect statistics for better join ordering
spark.sql("ANALYZE TABLE orders COMPUTE STATISTICS")
spark.sql("ANALYZE TABLE orders COMPUTE STATISTICS FOR COLUMNS customer_id, order_date")

# Enable CBO
spark.conf.set("spark.sql.cbo.enabled", "true")
spark.conf.set("spark.sql.cbo.joinReorder.enabled", "true")
```

---

## 📊 Reading Physical Plan Operators

| Operator | Meaning |
|----------|---------|
| `FileScan` | Reading from storage |
| `Filter` | Applying WHERE clause |
| `Project` | Selecting columns |
| `Exchange` | **Shuffle!** (expensive) |
| `Sort` | Sorting data |
| `HashAggregate` | GroupBy with hashing |
| `SortMergeJoin` | Large table join |
| `BroadcastHashJoin` | Small table broadcast join |
| `BroadcastExchange` | Broadcasting table to all nodes |

### Understanding `*` Notation

```
*(2) HashAggregate
```

The `*(2)` means this operation is part of **WholeStageCodegen #2** - Spark generates efficient Java bytecode for this entire stage.

---

## 🎯 Interview Answer Framework

When asked about Catalyst:

> **What it is:**
> *"Catalyst is Spark's query optimizer that transforms logical queries into optimized physical execution plans."*

> **Key stages:**
> *"It goes through 4 stages: parsing, analysis (resolve names), logical optimization (pushdown, pruning), and physical planning (choose join strategies)."*

> **Key optimizations:**
> *"Main optimizations include:*
> - *Predicate pushdown (filter at storage level)*
> - *Projection pruning (read only needed columns)*
> - *Join reordering (minimize data movement)*
> - *Broadcast join selection (for small tables)"*

> **How to debug:**
> ```python
> df.explain(True)  # Shows all optimization stages
> ```

---

## 💡 Common Performance Issues in Plans

### Red Flag 1: Missing Predicate Pushdown

```
# BAD: Filter not pushed
Filter (order_date = '2024-01-15')
+- FileScan parquet [all columns]
    PushedFilters: []  ← Empty! Reading all data

# Cause: Usually a UDF or complex expression blocking pushdown
```

### Red Flag 2: Unnecessary Shuffle

```
# BAD: Shuffle before broadcast
Exchange hashpartitioning(id, 200)
+- BroadcastHashJoin

# The shuffle is unnecessary if we're broadcasting anyway
```

### Red Flag 3: Full Table Scan

```
# BAD: No partition pruning
FileScan parquet
    PartitionFilters: []  ← Not using partitions!
```

---

## 📖 Next Topic

Continue to [File Formats Deep Dive](./04-file-formats-deep-dive.md) to understand Parquet, Avro, and Delta.
