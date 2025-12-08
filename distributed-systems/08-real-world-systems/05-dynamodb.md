# Amazon DynamoDB

> The fully managed, massively scalable NoSQL database.

---

## 🛒 **Shopping Cart Origin**

DynamoDB evolved from the Amazon Dynamo paper (2007), designed to ensure:
> "The shopping cart must always be available"

---

## 🎯 What Makes DynamoDB Special

```mermaid
graph TB
    subgraph "DynamoDB Features"
        Managed[Fully Managed<br/>No servers]
        Scale[Auto-scaling]
        Global[Global Tables]
        Consistency[Tunable Consistency]
    end
```

---

## 🏗️ Architecture

### Partition Keys & Sort Keys

```mermaid
graph TB
    subgraph "Table: Orders"
        PK[Partition Key: user_id] --> Part1[Partition 1]
        PK --> Part2[Partition 2]
        PK --> Part3[Partition 3]
        
        Part1 --> SK[Sort Key: order_date]
    end
```

```
Primary Key = Partition Key + (optional) Sort Key
```

| Example | Partition Key | Sort Key |
|---------|---------------|----------|
| User orders | user_id | order_date |
| Game scores | game_id | score |
| IoT readings | device_id | timestamp |

---

## 🔧 Consistent Hashing + Sloppy Quorums

```mermaid
graph TB
    subgraph "DynamoDB Replication"
        W[Write Request]
        
        W --> N1[Node 1 ✅]
        W --> N2[Node 2 ✅]
        W --> N3[Node 3 ⏳]
        
        Note[Write succeeds when<br/>W nodes respond]
    end
```

---

## ⚖️ Consistency Options

```mermaid
graph LR
    EC[Eventually Consistent<br/>Fast, may be stale] 
    SC[Strongly Consistent<br/>Latest, slower]
```

| Type | Latency | Guarantee | Cost |
|------|---------|-----------|------|
| Eventually Consistent | Lower | May be stale | 1 RCU |
| Strongly Consistent | Higher | Latest value | 2 RCUs |

```python
# Eventually consistent (default)
response = table.get_item(Key={'id': '123'})

# Strongly consistent
response = table.get_item(
    Key={'id': '123'},
    ConsistentRead=True
)
```

---

## 🔄 DynamoDB Streams

```mermaid
graph LR
    Table[(DynamoDB Table)]
    Stream[DynamoDB Stream]
    Lambda[Lambda Function]
    
    Table -->|Changes| Stream
    Stream -->|Trigger| Lambda
```

**Use cases**: 
- Replicate to other regions
- Build real-time analytics
- Trigger downstream processes

---

## 🌍 Global Tables

```mermaid
graph TB
    subgraph "US-East"
        US[(Table)]
    end
    
    subgraph "EU-West"
        EU[(Table)]
    end
    
    subgraph "AP-South"
        AP[(Table)]
    end
    
    US <-->|Multi-master| EU
    EU <-->|Replication| AP
    US <-->|Replication| AP
```

**Multi-master**: Write to any region!

---

## 📊 Access Patterns

### Single Table Design

```mermaid
graph TB
    subgraph "Traditional (Multiple Tables)"
        Users[(Users)]
        Orders[(Orders)]
        Products[(Products)]
    end
    
    subgraph "DynamoDB (Single Table)"
        ST[(One Table<br/>PK: entity#id<br/>SK: type#data)]
    end
```

**DynamoDB tip**: Model for access patterns, not entities!

---

## 🔍 Query vs Scan

| Operation | Use | Performance |
|-----------|-----|-------------|
| **Query** | By partition key | Fast ✅ |
| **Scan** | Full table | Slow ❌ (avoid!) |

---

## 🔥 Real-World: Amazon Scale

```mermaid
graph TB
    subgraph "Amazon Prime Day 2023"
        Scale[89.2 million requests/second]
        Peak[Peak at checkout]
    end
```

---

## ✅ Key Takeaways

1. **Partition key** determines data distribution
2. **Sort key** enables range queries within partition
3. **Eventually consistent** by default (faster, cheaper)
4. **Strongly consistent** when needed (+cost)
5. **Design for access patterns**, not normalized models
6. **Avoid scans** — use queries with partition keys

| Remember | Analogy |
|----------|---------|
| Partition key | Filing cabinet drawer |
| Sort key | Folder order in drawer |
| Query vs Scan | Looking in right drawer vs searching all |

---

[← Previous: Spanner](./04-spanner.md) | [Next: Kubernetes →](./06-kubernetes.md)
