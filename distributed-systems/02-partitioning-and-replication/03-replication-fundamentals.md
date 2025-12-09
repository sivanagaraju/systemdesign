# Replication Fundamentals

> Storing copies of data on multiple nodes for availability and durability.

---

## 🎯 Why Replicate?

```mermaid
graph TB
    subgraph "Without Replication"
        S1[Single Node] -->|Crash!| X((Data Lost))
    end
    
    subgraph "With Replication"
        P[Primary] -->|Crash!| Dead((❌))
        R1[Replica 1] -->|Takes over| OK((✅ Data Safe))
        R2[Replica 2]
    end
```

**Benefits**:

| Benefit | Description |
|---------|-------------|
| **Availability** | System continues if nodes fail |
| **Durability** | Data survives hardware failures |
| **Performance** | Read from nearest replica |
| **Geo-distribution** | Serve users from local region |

---

## 🔄 The Core Challenge

```mermaid
graph TB
    W[Write Request] --> P[Primary<br/>X = 5]
    P --> R1[Replica 1<br/>X = ?]
    P --> R2[Replica 2<br/>X = ?]
    
    Q{When do replicas<br/>have X = 5?}
    
    style Q fill:#fff9c4
```

**The fundamental question**: How and when do we propagate updates to replicas?

---

## 📊 Two Replication Strategies

```mermaid
graph LR
    subgraph "Pessimistic"
        P1[Ensure all replicas<br/>are in sync]
        P2[Before acknowledging<br/>the write]
    end
    
    subgraph "Optimistic"
        O1[Allow temporary<br/>divergence]
        O2[Converge eventually]
    end
```

---

## 1️⃣ Pessimistic (Synchronous) Replication

> All replicas are updated **before** acknowledging write to client.

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Primary
    participant R1 as Replica 1
    participant R2 as Replica 2
    
    C->>P: Write X=5
    P->>P: Write locally
    P->>R1: Replicate X=5
    P->>R2: Replicate X=5
    R1->>P: ACK
    R2->>P: ACK
    P->>C: Success ✅
    
    Note over C,R2: All replicas in sync<br/>before client sees success
```

### Characteristics

| Aspect | Description |
|--------|-------------|
| Consistency | ✅ Strong — read any replica, get latest |
| Durability | ✅ High — survives any single failure |
| Latency | ❌ Higher — wait for slowest replica |
| Availability | ❌ Lower — any replica down blocks writes |

### Real-World: PostgreSQL Synchronous Replication

```sql
-- Configure synchronous replication
synchronous_standby_names = 'replica1, replica2'
synchronous_commit = on
```

---

## 2️⃣ Optimistic (Asynchronous) Replication

> Write acknowledged **immediately**, replicate in background.

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Primary
    participant R1 as Replica 1
    participant R2 as Replica 2
    
    C->>P: Write X=5
    P->>P: Write locally
    P->>C: Success ✅
    
    Note over C: Client continues
    
    P-->>R1: Replicate X=5 (async)
    P-->>R2: Replicate X=5 (async)
    
    Note over P,R2: Replicas may lag behind
```

### Characteristics

| Aspect | Description |
|--------|-------------|
| Consistency | ⚠️ Eventual — replicas may return stale data |
| Durability | ⚠️ Risk — data loss if primary fails before replication |
| Latency | ✅ Low — no waiting for replicas |
| Availability | ✅ High — replica issues don't block writes |

### Real-World: MySQL Async Replication

```mermaid
graph TB
    subgraph "MySQL Replication"
        M[Master]
        BL[Binary Log]
        S1[Slave 1]
        S2[Slave 2]
        
        M --> BL
        BL -->|Relay| S1
        BL -->|Relay| S2
    end
    
    Note[Slaves may lag<br/>seconds to minutes]
```

---

## ⚖️ Comparison

```mermaid
graph LR
    subgraph "Trade-off Spectrum"
        Sync[Synchronous<br/>Strong consistency<br/>Higher latency]
        Async[Asynchronous<br/>Eventual consistency<br/>Lower latency]
    end
    
    Sync -- "Pick your trade-off" --- Async
```

| Property | Synchronous | Asynchronous |
|----------|-------------|--------------|
| Consistency | Strong | Eventual |
| Write latency | Higher | Lower |
| Durability | Higher | Lower |
| Availability | Lower (needs quorum) | Higher |
| Complexity | Higher | Lower |

---

## 🔥 Real-World Incident: GitHub 2018 Outage

**What happened**:

1. Network partition isolated primary MySQL
2. Cluster promoted a replica to primary
3. Old primary came back — had writes not yet replicated!
4. Data divergence between two "primaries"

**Impact**: 24 hours of degraded service, some data loss

**Lesson**: Async replication means potential data loss during failover.

```mermaid
graph TB
    subgraph "The Problem"
        P1[Old Primary<br/>Has writes A, B, C]
        P2[New Primary<br/>Has writes A, B]
        
        Conflict((Conflict!<br/>What happened to C?))
    end
    
    P1 --- Conflict
    P2 --- Conflict
    
    style Conflict fill:#f44336,color:#fff
```

---

## 🎚️ Semi-Synchronous: A Middle Ground

Some systems offer "semi-synchronous" replication:

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Primary
    participant R1 as Replica 1 (sync)
    participant R2 as Replica 2 (async)
    
    C->>P: Write X=5
    P->>P: Write locally
    P->>R1: Replicate X=5
    P-->>R2: Replicate X=5 (async)
    R1->>P: ACK
    P->>C: Success ✅
    
    Note over R2: May still be behind
```

**At least one replica** confirms synchronously, others async.

---

## ✅ Key Takeaways

1. **Pessimistic replication** = Strong consistency, higher latency, lower availability
2. **Optimistic replication** = Eventual consistency, lower latency, risk of data loss
3. **Semi-synchronous** = Compromise between the two
4. **All replication has trade-offs** — no free lunch!
5. **GitHub incident** shows real-world impact of async replication

---

[← Previous: Partitioning Algorithms](./02-partitioning-algorithms.md) | [Next: Primary-Backup Replication →](./04-primary-backup-replication.md)
