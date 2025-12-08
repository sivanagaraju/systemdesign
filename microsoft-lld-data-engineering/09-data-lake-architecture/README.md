# 09 - Data Lake Architecture

> **Modern data lake design patterns**

---

## 🏗️ Medallion Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   BRONZE    │───►│   SILVER    │───►│    GOLD     │
│   (Raw)     │    │  (Cleaned)  │    │ (Aggregated)│
└─────────────┘    └─────────────┘    └─────────────┘
     │                   │                   │
     ▼                   ▼                   ▼
 - Raw ingestion    - Deduplication     - Business logic
 - Schema on read   - Type casting      - Aggregations
 - Append only      - Data quality      - Dimensional models
```

---

## 🔑 Key Topics

| Topic | Description |
|-------|-------------|
| **Schema Evolution** | Adding columns, handling schema drift |
| **Data Versioning** | Delta Lake time travel |
| **Quality Gates** | Validation between layers |

---

## 💡 Schema Evolution (Delta Lake)

```python
# Enable schema evolution on write
df.write \
    .format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .save("/delta/table")
```

---

## ⏰ Time Travel

```python
# Read specific version
spark.read.format("delta") \
    .option("versionAsOf", 5) \
    .load("/delta/table")

# Restore previous version
spark.sql("RESTORE TABLE my_table TO VERSION AS OF 5")
```
