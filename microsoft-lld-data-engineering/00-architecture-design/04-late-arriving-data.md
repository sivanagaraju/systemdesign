# Late-Arriving Data Handling

> **THE critical scenario** - Products not arrived, but Orders came

## The Core Question

*"Orders data arrived, but the product_id doesn't exist in Products dimension yet. What do you do?"*

---

## 🔴 The Problem

```
Timeline:
─────────────────────────────────────────────────────────────────►

  10:00 AM                    10:30 AM                  11:00 AM
     │                           │                         │
     ▼                           ▼                         ▼
┌──────────┐              ┌──────────┐              ┌──────────┐
│  Orders  │              │ Products │              │  Join    │
│  Arrived │              │ Arrived  │              │  Runs    │
│          │              │          │              │          │
│ order_id │              │product_id│              │   ???    │
│product_id│              │   name   │              │          │
└──────────┘              └──────────┘              └──────────┘

Order file has product_id = 12345
But Products file hasn't arrived yet!

If we join now → Order is DROPPED or has NULL product name!
```

---

## 🏗️ Architecture Solutions

### Solution 1: Staging + Retry Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     STAGING + RETRY PATTERN                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐        ┌─────────────┐                                     │
│  │   ORDERS    │        │  PRODUCTS   │                                     │
│  │   (fact)    │        │   (dim)     │                                     │
│  └──────┬──────┘        └──────┬──────┘                                     │
│         │                      │                                             │
│         ▼                      ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         JOIN PROCESS                                    │ │
│  │                                                                         │ │
│  │   SELECT o.*, p.product_name                                            │ │
│  │   FROM orders o                                                         │ │
│  │   LEFT JOIN products p ON o.product_id = p.product_id                   │ │
│  └───────────────────────────────┬────────────────┬───────────────────────┘ │
│                                  │                │                          │
│              ┌───────────────────┘                └───────────────────┐      │
│              ▼                                                        ▼      │
│  ┌───────────────────────┐                            ┌───────────────────┐  │
│  │   MATCHED ORDERS      │                            │  UNMATCHED ORDERS │  │
│  │   (product found)     │                            │  (product missing)│  │
│  │                       │                            │                   │  │
│  │   → Write to Gold     │                            │   → Write to      │  │
│  │                       │                            │     STAGING TABLE │  │
│  └───────────────────────┘                            └─────────┬─────────┘  │
│                                                                 │            │
│                                                    ┌────────────┘            │
│                                                    ▼                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                    RETRY JOB (runs hourly)                               ││
│  │                                                                          ││
│  │   1. Read staging table                                                  ││
│  │   2. Join with latest Products dimension                                 ││
│  │   3. If matched → Move to Gold, delete from staging                      ││
│  │   4. If still unmatched → Keep in staging                                ││
│  │   5. After N retries → Move to dead letter / alert                       ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Code Implementation:**

```python
# Main processing job
def process_orders_with_late_arriving_dims(spark):
    orders_df = spark.read.format("delta").load("/silver/orders")
    products_df = spark.read.format("delta").load("/silver/products")
    
    # Left join to identify matched/unmatched
    joined = orders_df.join(
        products_df,
        on="product_id",
        how="left"
    )
    
    # Split into matched and unmatched
    matched = joined.filter(col("product_name").isNotNull())
    unmatched = joined.filter(col("product_name").isNull())
    
    # Write matched to Gold
    matched.write.format("delta").mode("append") \
        .save("/gold/enriched_orders")
    
    # Write unmatched to staging for retry
    if unmatched.count() > 0:
        unmatched.select("order_id", "product_id", "order_date", 
                        lit(current_timestamp()).alias("staged_at"),
                        lit(0).alias("retry_count")) \
            .write.format("delta").mode("append") \
            .save("/staging/unmatched_orders")

# Retry job (runs hourly)
def retry_unmatched_orders(spark):
    staging = spark.read.format("delta").load("/staging/unmatched_orders")
    products = spark.read.format("delta").load("/silver/products")
    
    # Try to match again
    retry_joined = staging.join(products, on="product_id", how="left")
    
    now_matched = retry_joined.filter(col("product_name").isNotNull())
    still_unmatched = retry_joined.filter(col("product_name").isNull())
    
    # Move matched to Gold
    if now_matched.count() > 0:
        now_matched.write.format("delta").mode("append") \
            .save("/gold/enriched_orders")
        # Delete from staging (use MERGE for Delta)
    
    # Increment retry count for still unmatched
    # Move to DLQ after max retries
```

---

### Solution 2: Unknown Member Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                  UNKNOWN MEMBER PATTERN                          │
│                                                                  │
│  Products Dimension:                                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ product_id │ product_name        │ category   │ status     │ │
│  ├────────────┼────────────────────┼────────────┼────────────┤ │
│  │     -1     │ Unknown Product     │ Unknown    │ PLACEHOLDER│ │  ← Default row
│  │     101    │ iPhone 15           │ Electronics│ ACTIVE     │ │
│  │     102    │ MacBook Pro         │ Electronics│ ACTIVE     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  When product_id not found:                                      │
│  JOIN with COALESCE(product_id, -1) → Gets "Unknown Product"    │
│                                                                  │
│  Later: Backfill job updates orders when product arrives         │
└─────────────────────────────────────────────────────────────────┘
```

---

### Solution 3: Event-Time Based Windowing

```
┌─────────────────────────────────────────────────────────────────┐
│               EVENT-TIME WINDOWING                               │
│                                                                  │
│  Don't process immediately!                                      │
│  Wait for a window where ALL data should have arrived.           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Time Window: 10:00 - 11:00 AM                               ││
│  │                                                               ││
│  │  Processing starts at: 11:30 AM (30 min buffer)              ││
│  │                                                               ││
│  │  By 11:30, both Orders AND Products for that hour            ││
│  │  should have arrived.                                        ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  Streaming version (watermarking):                               │
│  .withWatermark("event_time", "30 minutes")                     │
│  .groupBy(window("event_time", "1 hour"))                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Interview Answer Framework

> *"For late-arriving dimensions, I'd implement a Staging + Retry pattern:*
>
> 1. *Process with LEFT JOIN immediately*
> 2. *Matched records → Gold layer*
> 3. *Unmatched records → Staging table with timestamp*
> 4. *Hourly retry job re-attempts join with latest dimensions*
> 5. *After N retries (e.g., 24 hours) → Move to DLQ and alert*
>
> *Alternative: Unknown member pattern where dimension has a default -1 row, and backfill job updates later."*

---

## 📖 Next Topic

Continue to [Petabyte-Scale Patterns](./05-petabyte-scale-patterns.md) for distributed data handling.
