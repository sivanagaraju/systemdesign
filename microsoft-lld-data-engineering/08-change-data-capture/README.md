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

**Deep Dive Documentation:**

*   **[01 - CDC Fundamentals](01-cdc-fundamentals.md)**: The "Water" analogy, Pull vs. Push, and core concepts.
*   **[02 - System Design & Patterns](02-cdc-system-design.md)**: Ordering, deduplication, schema drift, and fault tolerance at a Staff level.
*   **[03 - Microsoft Ecosystem](03-microsoft-cdc-ecosystem.md)**: SQL Server CDC, Cosmos DB Change Feed, and ADF.
*   **[04 - CDC Cheat Sheet](04-cdc-cheat-sheet.md)**: The "BANANA" memory aid for interviews.
*   **[05 - CDC Architect Master Guide](05-cdc-architect-guide.md)**: The "FAANG Principal" level complete guide (10-point deep dive).
