# Azure Synapse Distribution Strategies

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Critical for Synapse roles)

## The Core Question

*"Design the distribution strategy for a 'Sales' fact table in Azure Synapse"*

---

## 📊 Distribution Types

| Type | How It Works | Best For |
|------|--------------|----------|
| **Hash** | Rows distributed by hash of column | Large fact tables, joins |
| **Round Robin** | Rows distributed evenly | Fast loading, no joins |
| **Replicated** | Full copy on each node | Small dimension tables |

---

## 🔄 Hash Distribution

```sql
CREATE TABLE fact_sales
WITH (
    DISTRIBUTION = HASH(customer_id),  -- Distribution key
    CLUSTERED COLUMNSTORE INDEX        -- Compression
)
AS SELECT * FROM staging_sales;
```

```
Node 1:                  Node 2:                  Node 3:
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│ Customers A-H   │      │ Customers I-P   │      │ Customers Q-Z   │
│ (hash 0-19)     │      │ (hash 20-39)    │      │ (hash 40-59)    │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

**When to use:**
- Large fact tables (> 60M rows)
- Tables frequently joined with others
- Choose high-cardinality column with even distribution

---

## ⭕ Round Robin (Default)

```sql
CREATE TABLE staging_sales
WITH (
    DISTRIBUTION = ROUND_ROBIN,
    HEAP  -- No index for fast loading
)
```

**When to use:**
- Staging tables (loading only)
- No analytics queries directly on table
- Maximum load performance

---

## 📋 Replicated

```sql
CREATE TABLE dim_product
WITH (
    DISTRIBUTION = REPLICATE,
    CLUSTERED COLUMNSTORE INDEX
)
```

```
Node 1:           Node 2:           Node 3:
┌──────────┐      ┌──────────┐      ┌──────────┐
│ ALL      │      │ ALL      │      │ ALL      │
│ Products │      │ Products │      │ Products │
│ (copy)   │      │ (copy)   │      │ (copy)   │
└──────────┘      └──────────┘      └──────────┘

No data movement for joins!
```

**When to use:**
- Small dimension tables (< 2GB)
- Frequently joined tables
- Trade-off: Extra storage, no shuffle

---

## 🔗 Join Optimization

For optimal joins, use **same hash key**:

```sql
-- Both tables distributed by customer_id
CREATE TABLE fact_sales
WITH (DISTRIBUTION = HASH(customer_id));

CREATE TABLE dim_customer
WITH (DISTRIBUTION = HASH(customer_id));

-- Join is local (no data movement!)
SELECT f.*, c.customer_name
FROM fact_sales f
JOIN dim_customer c ON f.customer_id = c.customer_id;
```

---

## 🎯 Interview Answer: Sales Fact Table

> *"For a Sales fact table, I'd use:*
>
> **Distribution:** `HASH(customer_id)`
> - *High cardinality (millions of customers)*
> - *Frequently joined with dim_customer*
> - *Collocated joins - no shuffle*
>
> **For dimensions:**
> - *`dim_product` → REPLICATE (< 2GB, frequently joined)*
> - *`dim_customer` → HASH(customer_id) (large, join-aligned)*
> - *`dim_date` → REPLICATE (small, every query joins it)*"

---

## 📖 Next Section

Move to [06 - Interview Scenarios](../06-interview-scenarios/README.md) for practice problems.
