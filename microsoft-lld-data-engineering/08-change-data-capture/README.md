# 08 - Change Data Capture (CDC)

> **Tracking and propagating data changes in distributed systems**

---

## 📚 Key Topics

| Topic | Description |
|-------|-------------|
| **Log-based CDC** | Read database transaction logs (Debezium, SQL Server CDC) |
| **Query-based CDC** | Polling with timestamps/versions |
| **MERGE/Upsert** | Delta Lake `MERGE INTO` patterns |
| **Incremental Load** | High/low watermark patterns |
| **Exactly-Once Processing** | Idempotent sinks and deduplication |
| **Schema Drift** | Handling upstream schema changes |

---

## 📁 Documentation

| File | Description |
|:-----|:------------|
| **[03 - Microsoft Ecosystem](03-microsoft-cdc-ecosystem.md)** | SQL Server CDC, Cosmos DB Change Feed, ADF, Debezium on Azure |
| **[05 - CDC Architect Master Guide](05-cdc-architect-guide.md)** | Complete deep-dive covering all CDC concepts, patterns, code snippets, and interview prep |

---

## 🔑 Quick Reference

### The "BANANA" Mnemonic

- **B**ackfill - Handle historical data without killing the source (Snapshotting)
- **A**t-Least-Once - Design for duplicate messages (Idempotency)
- **N**ear Real-time - Define latency SLA (Log-based vs Query-based)
- **A**lter Schema - Handle schema drift gracefully
- **N**o Locks - Zero impact on production database
- **A**sync - Decouple source and sink with a message broker

### Log-Based vs Query-Based CDC

| Approach | Pros | Cons |
|----------|------|------|
| **Log-based** | Real-time, complete history, captures deletes | Requires DB access, complex ops |
| **Query-based** | Simple, no special access | Misses deletes, polling overhead |

---

> **Recommended Path**: Start with the [05 - CDC Architect Master Guide](05-cdc-architect-guide.md) for a comprehensive understanding, then reference [03 - Microsoft Ecosystem](03-microsoft-cdc-ecosystem.md) for Azure-specific implementations.
