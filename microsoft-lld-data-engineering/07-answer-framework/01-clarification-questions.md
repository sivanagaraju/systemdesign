# Clarification Questions

> **What to ask before designing**

Asking good questions shows maturity and prevents wasted effort.

---

## 🎯 Questions by Topic

### Schema Design

- *"Are we optimizing for read performance (analytics) or write performance (ingestion)?"*
- *"What are the typical query patterns - aggregations or point lookups?"*
- *"How often do dimension attributes change?"*
- *"What's the data retention policy?"*

### Pipeline Design

- *"What's the SLA for data freshness - real-time or daily batch?"*
- *"What's the expected volume - rows per day/hour?"*
- *"Are sources reliable, or do we need to handle failures?"*
- *"Is idempotency required - can pipeline safely rerun?"*

### Spark Optimization

- *"What's the cluster size - how many executors?"*
- *"What are the table sizes for the join?"*
- *"Is data already partitioned or sorted?"*
- *"Are there known data skew issues?"*

### API Integration

- *"What are the rate limits?"*
- *"Is authentication required - OAuth, API key?"*
- *"How should we handle partial failures?"*
- *"Is the API idempotent?"*

---

## 💡 Why This Matters

| Without Clarification | With Clarification |
|-----------------------|-------------------|
| Design for wrong use case | Targeted solution |
| Miss critical requirements | Cover all needs |
| Waste time redesigning | Efficient design |
| Look junior | Show experience |

---

## 📖 Next

Continue to [Edge Cases Checklist](./02-edge-cases-checklist.md).
