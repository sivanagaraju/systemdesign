# Multi-Primary Replication

> Multiple nodes accept writes — higher availability, but conflict resolution required.

---

## 🎯 Architecture

```mermaid
graph TB
    C1[Client 1] -->|Write| P1[Primary 1]
    C2[Client 2] -->|Write| P2[Primary 2]
    C3[Client 3] -->|Write| P3[Primary 3]
    
    P1 <-->|Sync| P2
    P2 <-->|Sync| P3
    P1 <-->|Sync| P3
    
    style P1 fill:#4caf50,color:#fff
    style P2 fill:#4caf50,color:#fff
    style P3 fill:#4caf50,color:#fff
```

**Also known as**: Multi-Master, Active-Active

---

## ⚖️ Comparison with Primary-Backup

| Aspect | Primary-Backup | Multi-Primary |
|--------|----------------|---------------|
| Write nodes | 1 | Multiple |
| Read scaling | ✅ Yes | ✅ Yes |
| Write scaling | ❌ No | ✅ Yes |
| Failover needed | ✅ Yes | ❌ No |
| Conflicts | ❌ None | ✅ Possible |
| Complexity | Lower | Higher |

---

## 💥 The Conflict Problem

When two nodes accept writes for the **same data** concurrently:

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant P1 as Primary 1
    participant P2 as Primary 2
    participant C2 as Client 2
    
    C1->>P1: SET name = Alice
    C2->>P2: SET name = Bob
    
    Note over P1,P2: Both writes accepted!
    
    P1->>P2: Sync: name = Alice
    P2->>P1: Sync: name = Bob
    
    Note over P1,P2: CONFLICT!<br/>Which value wins?
```

---

## 🔧 Conflict Resolution Strategies

### 1. Last-Write-Wins (LWW)

```mermaid
graph LR
    W1[Write: name=Alice<br/>timestamp: 100] 
    W2[Write: name=Bob<br/>timestamp: 105]
    
    W1 --> Conflict{Conflict}
    W2 --> Conflict
    
    Conflict --> Winner[Bob wins!<br/>Higher timestamp]
    
    style Winner fill:#4caf50,color:#fff
```

**Implementation**:

```python
def resolve(write1, write2):
    return write1 if write1.timestamp > write2.timestamp else write2
```

**Problems**:

- Clock skew can cause "earlier" writes to win
- Silent data loss — user's write disappears

**Used by**: Cassandra, DynamoDB (by default)

---

### 2. Expose to Application

Return **all versions** and let the application/user decide.

```mermaid
sequenceDiagram
    participant C as Client
    participant DB as Database
    
    C->>DB: GET shopping_cart
    DB->>C: Version 1: [iPhone]<br/>Version 2: [iPad]
    
    Note over C: Application merges
    
    C->>DB: PUT shopping_cart = [iPhone, iPad]
```

**Used by**: Amazon DynamoDB (optional), Riak

---

### 3. Application-Defined Rules

```mermaid
graph TB
    subgraph "Custom Merge Logic"
        C1[Counter: +5]
        C2[Counter: +3]
        
        Merge[Merge: 5 + 3 = 8]
    end
```

**Example**: CRDTs (Conflict-free Replicated Data Types)

- **G-Counter**: Only grows, merge = take max from each node
- **PN-Counter**: Track increments and decrements separately
- **LWW-Register**: Last-write-wins for single values
- **OR-Set**: Observed-remove set for collections

---

### 4. Deterministic Resolution

Use a deterministic tie-breaker when timestamps are equal:

```python
def resolve(write1, write2):
    if write1.timestamp != write2.timestamp:
        return max(write1, write2, key=lambda w: w.timestamp)
    # Tie-breaker: node ID
    return max(write1, write2, key=lambda w: w.node_id)
```

---

## 🔥 Real-World: Cassandra Conflict Resolution

```mermaid
graph TB
    subgraph "Cassandra Write Path"
        W[Write: X=5, timestamp=100]
        N1[Node 1: X=5 @ 100]
        N2[Node 2: X=3 @ 95]
        N3[Node 3: X=5 @ 100]
    end
    
    Read[Read Request]
    Read --> N1
    Read --> N2
    Read --> N3
    
    Resolver[Coordinator:<br/>X=5 wins (timestamp 100)]
    
    N1 --> Resolver
    N2 --> Resolver
    N3 --> Resolver
```

**Cassandra approach**:

1. Writes go to multiple nodes with timestamps
2. On read, coordinator queries replicas
3. Latest timestamp wins
4. **Read repair**: Stale nodes get updated

---

## 🏢 Use Cases for Multi-Primary

### 1. Geographic Distribution

```mermaid
graph TB
    subgraph "US East"
        US[Primary US]
    end
    
    subgraph "Europe"
        EU[Primary EU]
    end
    
    subgraph "Asia"
        AS[Primary Asia]
    end
    
    US <-->|Async sync| EU
    EU <-->|Async sync| AS
    US <-->|Async sync| AS
    
    U1[User US] --> US
    U2[User EU] --> EU
    U3[User Asia] --> AS
```

**Benefit**: Users write to nearest datacenter

### 2. High Availability

No need for failover — if one primary fails, others continue.

### 3. Collaborative Editing

Google Docs, Figma — multiple users edit simultaneously.

---

## ⚠️ When NOT to Use Multi-Primary

| Scenario | Problem |
|----------|---------|
| Banking transactions | Conflicts could mean money issues |
| Inventory management | Overselling due to concurrent updates |
| Unique constraints | Hard to enforce across nodes |
| Where ordering matters | No single source of truth |

---

## 📊 Multi-Primary Systems

| System | Conflict Resolution | Use Case |
|--------|---------------------|----------|
| Cassandra | LWW + Vector clocks | Time-series, high write volume |
| CockroachDB | Serializable (prevents conflicts) | OLTP with global distribution |
| Galera Cluster (MySQL) | Certification-based | HA MySQL |
| Riak | CRDTs, Siblings | Key-value at scale |

---

## ✅ Key Takeaways

1. **Multi-primary** allows writes to any node → higher availability and write scaling
2. **Conflicts are inevitable** when same data is written concurrently
3. **LWW is simple but lossy** — data silently disappears
4. **CRDTs** enable automatic, lossless merging for certain data types
5. **Choose carefully** — multi-primary adds complexity
6. **Best for**: geo-distributed, high-availability, eventual consistency acceptable

---

[← Previous: Primary-Backup](./04-primary-backup-replication.md) | [Next: Quorums →](./06-quorums.md)
