# Duplicate Records Handling

> **Senior Staff / Principal Interview Scenario**
>
> "Your source system sends the same order twice. How do you prevent duplicates in your data lake?"

---

## 🔴 The Real-World Scenario

> *"It's Friday 4 PM. Finance reports that revenue for today is showing $2.3M, but it should be ~$1.5M. Investigation reveals: a network blip caused the payment API to retry, and 30% of transactions were processed twice. Each duplicate added to the revenue total."*

**Business Impact**:

- Revenue reports off by millions
- Inventory counts incorrect
- Customer charged twice (refund nightmare)

**Root Cause**: **At-least-once delivery** without idempotent processing creates duplicates.

---

## 📚 Key Terminology

| Term | Definition | Example |
|:-----|:-----------|:--------|
| **Deduplication** | Process of identifying and removing duplicate records | Keep only one `order_id = 123` |
| **Dedup Key** | Column(s) that uniquely identify a logical record | `order_id` or `order_id + event_time` |
| **At-Least-Once** | Delivery guarantee that may produce duplicates | Kafka default |
| **Exactly-Once** | Delivery guarantee with no duplicates | Kafka transactions + idempotent sink |
| **Idempotent Operation** | Operation safe to apply multiple times | MERGE/upsert |
| **Stateful Dedup** | Remembering seen keys in streaming state | Requires watermark for bounded state |

---

## 🔬 Deep-Dive: Why Duplicates Happen

| Source | Why It Happens | Frequency |
|:-------|:---------------|:----------|
| **Network Retry** | Client didn't receive ACK, retries | Common |
| **Producer Retry** | Kafka producer retries on timeout | Common |
| **At-Least-Once** | Designed to never lose, may duplicate | By design |
| **Source Bug** | Application sends same event twice | Occasional |
| **Reprocessing** | Pipeline replays from checkpoint | Intentional |
| **File Re-upload** | Same file uploaded twice | Human error |

---

## 🏗️ Deduplication Architecture

```text
Source Duplicates:              What You Receive:
───────────────────────         ───────────────────────
- Network retry                 Order 123 at 10:00
- Producer retry                Order 123 at 10:01  ← DUPLICATE!
- At-least-once delivery        Order 123 at 10:02  ← DUPLICATE!
- Bug in source                 Order 124 at 10:03
```

```mermaid
flowchart TD
    subgraph Sources ["Ingestion Sources"]
        K[Kafka (At-Least-Once)]
        A[API (Retries)]
        F[Files (Re-sent)]
    end
    
    subgraph Bronze_Layer ["Bronze Layer (Raw History)"]
        Raw[Ingest All Records]
        Raw --> |"Contains Duplicates"| Bronze[(Bronze Delta)]
        
        style Bronze fill:#efdcd5,stroke:#5d4037
        Note["Preserve Audit Trail<br/>Ingest Everything"] -.-> Bronze
    end
    
    subgraph Silver_Layer ["Silver Layer (Deduplicated)"]
        Bronze --> Window{Window Rank}
        
        Window --> |"PARTITION BY id<br/>ORDER BY time DESC"| Rank[Assign Row Num]
        Rank --> |"Filter WHERE rn=1"| Clean[Unique Records]
        
        Clean --> Silver[(Silver Delta)]
        style Silver fill:#e0f2f1,stroke:#00695c
    end
    
    K --> Raw
    A --> Raw
    F --> Raw
```

---

## 🔧 Code Implementation

### Method 1: dropDuplicates (Simple)

```python
# Drop exact duplicates
df_deduped = df.dropDuplicates()

# Drop duplicates by key columns only (keeps first occurrence)
df_deduped = df.dropDuplicates(["order_id"])

# Problem: "first" is arbitrary - which duplicate is kept is undefined!
```

### Method 2: Window Function (Precise Control)

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number, col

# Define window: partition by business key, order by event_time DESC
window_spec = Window.partitionBy("order_id") \
                    .orderBy(col("event_time").desc())

# Assign row numbers (1 = most recent)
df_with_rn = df.withColumn("rn", row_number().over(window_spec))

# Keep only the most recent version of each order
df_deduped = df_with_rn.filter(col("rn") == 1).drop("rn")
```

### Method 3: MERGE (Delta Lake - Best for Updates)

```python
from delta.tables import DeltaTable

# Get existing silver table
silver_table = DeltaTable.forPath(spark, "/silver/orders")

# New records from bronze (may contain duplicates)
new_records = spark.read.format("delta").load("/bronze/orders") \
    .filter(col("ingestion_date") == current_date())

# Deduplicate new records first
new_deduped = new_records \
    .withColumn("rn", row_number().over(
        Window.partitionBy("order_id").orderBy(col("event_time").desc())
    )) \
    .filter(col("rn") == 1) \
    .drop("rn")

# MERGE: Update if newer, Insert if new
silver_table.alias("target").merge(
    new_deduped.alias("source"),
    condition="target.order_id = source.order_id"
).whenMatchedUpdate(
    condition="source.event_time > target.event_time",  # Only if newer
    set={"*": "source.*"}
).whenNotMatchedInsert(
    values={"*": "source.*"}
).execute()
```

### Method 4: Streaming Deduplication

```python
# For streaming: use dropDuplicates with watermark
stream_df = spark.readStream.format("delta").load("/bronze/orders")

# Dedup within watermark window
deduped_stream = stream_df \
    .withWatermark("event_time", "10 minutes") \
    .dropDuplicates(["order_id", "event_time"])

# Write to silver
deduped_stream.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/checkpoints/orders_dedup") \
    .start("/silver/orders")
```

---

## 📊 Deduplication Key Selection

| Use Case | Dedup Key | Order By |
|----------|-----------|----------|
| Orders | `order_id` | `event_time DESC` (latest wins) |
| Customer updates | `customer_id` | `modified_at DESC` |
| IoT events | `device_id, event_time` | `ingestion_time DESC` |
| CDC records | `primary_key, operation_timestamp` | `operation_timestamp DESC` |

---

## ⚠️ Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Dedup at Bronze | Lose audit trail | Keep raw at Bronze, dedup at Silver |
| `dropDuplicates()` without key | Undefined which kept | Use explicit key columns |
| No ordering | Random record kept | Always ORDER BY event_time |
| Dedup before watermark (streaming) | High memory | Apply watermark first |

---

## 🎯 Interview Questions

| Question | Expected Answer |
|----------|----------------|
| *"How do you handle duplicate records?"* | Dedup at Bronze→Silver using window function with ROW_NUMBER() |
| *"Which duplicate do you keep?"* | Keep the one with latest event_time (ORDER BY DESC, take rn=1) |
| *"Why not dedup at Bronze?"* | Bronze is raw - keep for auditing, debugging, reprocessing |
| *"How to dedup in streaming?"* | `dropDuplicates()` with watermark for bounded state |
| *"What's the dedup key?"* | Business key (order_id) + optionally event_time |

---

## 📖 Next Scenario

Continue to [Schema Drift Handling](./09-schema-drift.md).
