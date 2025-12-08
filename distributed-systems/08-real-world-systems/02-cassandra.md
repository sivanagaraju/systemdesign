# Apache Cassandra

> The highly available, eventually consistent, wide-column database.

---

## 🎯 What is Cassandra?

```mermaid
graph TB
    subgraph "Cassandra Ring"
        N1[Node 1]
        N2[Node 2]
        N3[Node 3]
        N4[Node 4]
        N5[Node 5]
        N6[Node 6]
        
        N1 --- N2
        N2 --- N3
        N3 --- N4
        N4 --- N5
        N5 --- N6
        N6 --- N1
    end
    
    Client((Client)) --> N2
    Client --> N5
```

**Cassandra is a masterless, peer-to-peer distributed database** designed for high availability.

---

## 🏗️ Architecture

### Consistent Hashing Ring

```mermaid
graph TB
    subgraph "Token Ring (0-100)"
        N1[Node 1<br/>Tokens: 0-15]
        N2[Node 2<br/>Tokens: 16-32]
        N3[Node 3<br/>Tokens: 33-49]
        N4[Node 4<br/>Tokens: 50-66]
        N5[Node 5<br/>Tokens: 67-83]
        N6[Node 6<br/>Tokens: 84-100]
    end
    
    Key[Key: user123<br/>Token: 42] -->|Belongs to| N3
```

### Data Distribution

```mermaid
graph TB
    subgraph "Replication Factor = 3"
        Key[Key: user123] --> N3[Node 3<br/>Primary]
        Key --> N4[Node 4<br/>Replica]
        Key --> N5[Node 5<br/>Replica]
    end
```

---

## 🔧 Key Concepts

### Primary Key Structure

```sql
CREATE TABLE users (
    user_id UUID,
    event_time TIMESTAMP,
    event_type TEXT,
    data TEXT,
    PRIMARY KEY ((user_id), event_time, event_type)
);
```

```mermaid
graph LR
    PK[Primary Key] --> Partition[Partition Key<br/>user_id]
    PK --> Clustering[Clustering Columns<br/>event_time, event_type]
    
    Partition --> |Determines| Node[Which node?]
    Clustering --> |Determines| Order[Order within partition]
```

### Wide Rows

```mermaid
graph TB
    subgraph "user_id = 123"
        Row[Wide Row]
        C1[event_time:01<br/>type:login]
        C2[event_time:02<br/>type:click]
        C3[event_time:03<br/>type:purchase]
        C4[... millions of columns]
        
        Row --> C1
        Row --> C2
        Row --> C3
        Row --> C4
    end
```

---

## ⚖️ Consistency Levels

```mermaid
graph TB
    subgraph "Tunable Consistency"
        ONE[ONE<br/>Fastest, weakest]
        QUORUM[QUORUM<br/>Balanced]
        ALL[ALL<br/>Slowest, strongest]
    end
```

| Level | Read | Write | Guarantee |
|-------|------|-------|-----------|
| ONE | 1 replica | 1 replica | Eventual |
| QUORUM | N/2+1 | N/2+1 | Strong* |
| ALL | All replicas | All replicas | Strongest |

*Strong consistency when: `R + W > N`

---

## 🔄 Write Path

```mermaid
sequenceDiagram
    participant C as Client
    participant Coord as Coordinator
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3
    
    C->>Coord: Write (CL=QUORUM)
    Coord->>N1: Write
    Coord->>N2: Write
    Coord->>N3: Write (async)
    
    N1->>Coord: ACK
    N2->>Coord: ACK
    Note over Coord: Quorum met!
    Coord->>C: Success
    
    N3->>Coord: ACK (late)
```

### Internal Write Flow

```mermaid
graph TB
    Write[Write Request] --> CL[Commit Log<br/>Append only]
    CL --> MT[Memtable<br/>In-memory]
    MT -->|Flush| SST[SSTable<br/>On disk]
```

---

## 🔄 Read Path

```mermaid
graph TB
    Read[Read Request] --> BF[Bloom Filter<br/>Skip SSTables?]
    BF --> MT[Memtable]
    BF --> SST[SSTables]
    
    MT --> Merge[Merge Results]
    SST --> Merge
    
    Merge --> Return[Return to Client]
```

---

## ⚠️ Anti-Patterns

### ❌ Unbounded Partitions

```sql
-- BAD: All data for user in one partition
PRIMARY KEY (user_id)

-- GOOD: Bucket by time
PRIMARY KEY ((user_id, month), event_time)
```

### ❌ Secondary Indexes on High-Cardinality

```sql
-- BAD: Index on unique values
CREATE INDEX ON users(email);

-- Use: Materialized view or denormalize
```

---

## 🔥 Real-World: Netflix

```mermaid
graph TB
    subgraph "Netflix Cassandra Usage"
        Scale[2500+ instances]
        Data[12 million writes/sec]
        Size[420 TB data]
    end
```

**Use cases**:
- User viewing history
- Movie metadata
- A/B test data
- Real-time analytics

---

## 📊 When to Use Cassandra

| Good For | Not Good For |
|----------|--------------|
| ✅ High write throughput | ❌ Complex joins |
| ✅ Time-series data | ❌ Ad-hoc queries |
| ✅ Geo-distributed | ❌ Strong consistency required |
| ✅ Known query patterns | ❌ Small datasets |

---

## 📋 Comparison: Cassandra vs Others

| Feature | Cassandra | MongoDB | DynamoDB |
|---------|-----------|---------|----------|
| Data model | Wide-column | Document | Key-value |
| Consistency | Tunable | Tunable | Tunable |
| Scaling | Masterless | Sharded | Managed |
| Query language | CQL | MQL | API |

---

## ✅ Key Takeaways

1. **Masterless architecture** — no single point of failure
2. **Consistent hashing** for data distribution
3. **Tunable consistency** — choose per query
4. **Write-optimized** — LSM tree structure
5. **Design for your queries** — denormalization is key
6. **Used by**: Netflix, Apple, Instagram, Uber

---

[← Previous: Kafka](./01-kafka.md) | [Next: ZooKeeper →](./03-zookeeper.md)
