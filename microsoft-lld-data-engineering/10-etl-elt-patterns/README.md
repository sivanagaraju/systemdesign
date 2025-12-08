# 10 - ETL/ELT Design Patterns

> **Processing patterns for data pipelines**

---

## ⚖️ ETL vs ELT

| Pattern | Transform Location | Best For |
|---------|-------------------|----------|
| **ETL** | Before loading (Spark) | Complex transformations |
| **ELT** | After loading (SQL) | Modern cloud DWs (Synapse) |

---

## 🔄 Batch vs Streaming

### Lambda Architecture
```
┌─────────────┐
│   Source    │
└──────┬──────┘
       │
 ┌─────┴─────┐
 │           │
 ▼           ▼
Batch      Streaming
 │           │
 └─────┬─────┘
       ▼
 Serving Layer
```

### Kappa Architecture
```
Source → Streaming Only → Serving Layer
(Replay from Kafka for batch use cases)
```

---

## 📄 Data Contracts

Define expectations between producer and consumer:

```yaml
contract:
  name: orders_v1
  owner: sales-team
  schema:
    - name: order_id
      type: string
      required: true
    - name: amount
      type: decimal
      required: true
  freshness: "< 1 hour"
  quality:
    - "amount >= 0"
    - "order_id IS NOT NULL"
```

---

## ⚡ Event-Driven Patterns

```python
# Trigger pipeline on file arrival (ADF example)
{
    "trigger": {
        "type": "BlobEventsTrigger",
        "events": ["Microsoft.Storage.BlobCreated"],
        "scope": "/subscriptions/.../containers/raw"
    }
}
```
