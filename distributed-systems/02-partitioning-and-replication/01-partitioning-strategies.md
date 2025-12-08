# Partitioning Strategies

> Splitting data across multiple nodes to achieve scalability.

---

## 🎯 Why Partition?

```mermaid
graph TB
    subgraph "Without Partitioning"
        S1[Single Node<br/>1TB Data<br/>⚠️ Limit reached]
    end
    
    subgraph "With Partitioning"
        P1[Node 1<br/>333GB]
        P2[Node 2<br/>333GB]
        P3[Node 3<br/>333GB]
    end
    
    S1 -.->|Split| P1
    S1 -.->|Split| P2
    S1 -.->|Split| P3
```

**Benefits**:
- **Scalability**: Handle more data than fits on one machine
- **Performance**: Parallel processing across nodes
- **Cost**: Use commodity hardware instead of expensive servers

---

## 📊 Two Types of Partitioning

```mermaid
graph LR
    subgraph "Original Table"
        O[ID | Name | Email | Age]
    end
    
    O -->|Vertical| V1[ID | Name]
    O -->|Vertical| V2[ID | Email | Age]
    
    O -->|Horizontal| H1[Rows 1-1000]
    O -->|Horizontal| H2[Rows 1001-2000]
```

---

## 1️⃣ Vertical Partitioning

> Split table by **columns** — different attributes on different nodes.

```mermaid
graph TB
    subgraph "Original Users Table"
        OT[id | name | email | avatar_blob | preferences_json]
    end
    
    subgraph "After Vertical Partitioning"
        T1[Core Service<br/>id | name | email]
        T2[Media Service<br/>id | avatar_blob]
        T3[Settings Service<br/>id | preferences_json]
    end
    
    OT --> T1
    OT --> T2
    OT --> T3
```

### Use Cases
- **Separate hot and cold data**: Frequently accessed vs rarely accessed columns
- **Different storage needs**: BLOBs in object storage, structured data in RDBMS
- **Microservices**: Each service owns its data

### Real-World Example: E-commerce

```mermaid
graph TB
    subgraph "Product Data Split"
        Core[Products Service<br/>id, name, price, category]
        Media[Media Service<br/>id, images, videos]
        Inventory[Inventory Service<br/>id, stock_count, warehouse_id]
        Reviews[Reviews Service<br/>product_id, rating, text]
    end
    
    style Core fill:#e3f2fd
    style Media fill:#fff3e0
    style Inventory fill:#e8f5e9
    style Reviews fill:#fce4ec
```

### Pros & Cons

| Pros | Cons |
|------|------|
| ✅ Optimize storage per column type | ❌ JOINs become expensive network calls |
| ✅ Independently scale each partition | ❌ Increases system complexity |
| ✅ Better caching (smaller row size) | ❌ Transactions across partitions are hard |

---

## 2️⃣ Horizontal Partitioning (Sharding)

> Split table by **rows** — different records on different nodes.

```mermaid
graph TB
    subgraph "Original Table: 10M Users"
        OT[All Users]
    end
    
    subgraph "After Sharding"
        S1[Shard 1<br/>Users A-F<br/>~3.3M rows]
        S2[Shard 2<br/>Users G-N<br/>~3.3M rows]
        S3[Shard 3<br/>Users O-Z<br/>~3.3M rows]
    end
    
    OT --> S1
    OT --> S2
    OT --> S3
```

### Choosing a Partition Key

The **partition key** determines which shard stores each row.

```mermaid
graph LR
    Data[User Record] --> Key[Partition Key<br/>e.g., user_id]
    Key --> Hash[Hash Function]
    Hash --> Shard[Shard Assignment]
```

**Good Partition Keys**:
| Key | Why Good |
|-----|----------|
| `user_id` | Evenly distributed, queries usually include it |
| `tenant_id` | Multi-tenant apps, isolates customer data |
| `region` | Geographic queries, data locality |

**Bad Partition Keys**:
| Key | Why Bad |
|-----|---------|
| `created_at` | Hot spots (all new data in one shard) |
| `country` | Uneven (most users in few countries) |
| `status` | Very few values, poor distribution |

---

## 🔥 Real-World: Instagram Sharding

Instagram sharded their PostgreSQL database by user_id:

```mermaid
graph TB
    subgraph "Instagram Sharding"
        Router[Shard Router]
        
        Router -->|user_id % 4 = 0| S0[Shard 0]
        Router -->|user_id % 4 = 1| S1[Shard 1]
        Router -->|user_id % 4 = 2| S2[Shard 2]
        Router -->|user_id % 4 = 3| S3[Shard 3]
    end
    
    Note[Each shard is a<br/>PostgreSQL instance]
```

**Key Decisions**:
- Partition key: `user_id` (most queries are user-centric)
- Generated IDs include shard ID for routing
- Cross-shard queries minimized by design

---

## ⚠️ Challenges of Sharding

### 1. Cross-Shard Queries

```mermaid
sequenceDiagram
    participant C as Client
    participant R as Router
    participant S1 as Shard 1
    participant S2 as Shard 2
    participant S3 as Shard 3
    
    C->>R: SELECT * WHERE age > 25
    R->>S1: Query
    R->>S2: Query
    R->>S3: Query
    S1->>R: Results
    S2->>R: Results
    S3->>R: Results
    R->>C: Merged Results
    
    Note over R: Scatter-Gather<br/>(expensive!)
```

### 2. Cross-Shard Transactions

```mermaid
graph TB
    TX[Transaction: Transfer $100<br/>from User A to User B]
    
    TX --> S1[Shard 1<br/>User A: -$100]
    TX --> S2[Shard 2<br/>User B: +$100]
    
    Note[Need 2PC or Saga<br/>for atomicity!]
    
    style Note fill:#fff9c4
```

### 3. Rebalancing

When adding/removing shards, data must move:

```mermaid
graph LR
    subgraph "Before: 3 Shards"
        B1[Shard 1: A-I]
        B2[Shard 2: J-R]
        B3[Shard 3: S-Z]
    end
    
    subgraph "After: 4 Shards"
        A1[Shard 1: A-F]
        A2[Shard 2: G-M]
        A3[Shard 3: N-S]
        A4[Shard 4: T-Z]
    end
    
    B1 -.->|Data moves| A1
    B1 -.->|Data moves| A2
    B2 -.->|Data moves| A2
    B2 -.->|Data moves| A3
```

---

## 📊 Comparison Summary

| Aspect | Vertical | Horizontal |
|--------|----------|------------|
| Split by | Columns | Rows |
| Use case | Different data types/access patterns | Scale beyond single node |
| Complexity | Moderate | High |
| Data locality | Same row still on one node | May need cross-shard queries |
| Examples | Microservices | Sharded databases |

---

## ✅ Key Takeaways

1. **Vertical partitioning** splits by columns — good for microservices
2. **Horizontal partitioning (sharding)** splits by rows — needed for scale
3. **Partition key choice is critical** — determines data distribution
4. **Sharding introduces complexity**: cross-shard queries, transactions, rebalancing
5. **Design your data model for your partition key** (denormalization helps)

---

[← Back to Module](./README.md) | [Next: Partitioning Algorithms →](./02-partitioning-algorithms.md)
