# 01 - Schema & Data Modeling

> **Core LLD Skill:** Designing efficient, scalable data models for analytics workloads

Schema design is often the **first LLD topic** in Microsoft Data Engineering interviews. They want to see you make trade-offs between query performance, storage efficiency, and maintainability.

---

## 📚 Topics in This Section

| File | Topic | Key Concepts |
|------|-------|--------------|
| [01-star-vs-snowflake-schema.md](./01-star-vs-snowflake-schema.md) | Dimensional Modeling | Star schema, Snowflake, when to use each |
| [02-slowly-changing-dimensions.md](./02-slowly-changing-dimensions.md) | SCD Handling | Type 1-6, implementation patterns |
| [03-partitioning-vs-bucketing.md](./03-partitioning-vs-bucketing.md) | Physical Layout | Partition pruning, small files problem |
| [04-cosmos-db-partition-key.md](./04-cosmos-db-partition-key.md) | NoSQL Design | Partition key selection, hot partitions |

---

## 🎯 Common Interview Questions

1. *"Design the schema to store telemetry data for Xbox gaming sessions"*
2. *"How would you partition this data in ADLS Gen2?"*
3. *"Choose the right partition key for this Cosmos DB collection"*
4. *"How do you track changes to customer address over time?"*

---

## 🔑 Key Trade-offs to Discuss

| Decision | Optimize For | Trade-off |
|----------|--------------|-----------|
| Star vs Snowflake | Query speed vs Storage | Denormalized = fast queries, more storage |
| Partitioning | Read pruning | Too granular = small files problem |
| Bucketing | Join performance | Write overhead, fixed bucket count |
| SCD Type | History tracking | Type 2 = more storage, complex queries |

---

## 🏗️ Schema Design Framework

When asked to design a schema, follow this flow:

```
1. Identify FACTS (events, transactions, measurements)
2. Identify DIMENSIONS (who, what, where, when)
3. Choose granularity (one row = one what?)
4. Decide partitioning strategy
5. Consider query patterns (OLAP vs OLTP)
6. Plan for history (SCD type)
```

---

## 📖 Start Here

Begin with [Star vs Snowflake Schema](./01-star-vs-snowflake-schema.md) to understand the foundation of dimensional modeling.
