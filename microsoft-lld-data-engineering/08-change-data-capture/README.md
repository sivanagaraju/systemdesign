# 08 - Change Data Capture (CDC)

> **Tracking and propagating data changes**

---

## 📚 Key Topics

| Topic | Description |
|-------|-------------|
| **Log-based CDC** | Read database transaction logs (Debezium) |
| **Query-based CDC** | Polling with timestamps/versions |
| **MERGE/Upsert** | Delta Lake `MERGE INTO` patterns |
| **Incremental Load** | High/low watermark patterns |

---

## 🔑 Quick Reference

### Log-Based vs Query-Based CDC

| Approach | Pros | Cons |
|----------|------|------|
| **Log-based** | Real-time, complete history | Requires DB access |
| **Query-based** | Simple, no special access | Misses deletes, polling overhead |

### Delta Lake MERGE Pattern

```python
target_table.alias("t").merge(
    source_df.alias("s"),
    "t.id = s.id"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()
```

### Incremental Load

```sql
-- Get last watermark
SELECT MAX(modified_date) FROM target_table;

-- Load only changes
SELECT * FROM source_table 
WHERE modified_date > @last_watermark;
```

---

**Full details:** See dedicated files in this folder for deep dives.
