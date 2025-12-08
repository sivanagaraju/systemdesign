# Primary-Backup Replication

> The most common replication pattern: one leader handles writes, followers handle reads.

---

## 🎯 Architecture

```mermaid
graph TB
    C[Client] -->|Writes| P[Primary/Leader]
    C -->|Reads| P
    C -->|Reads| F1[Follower 1]
    C -->|Reads| F2[Follower 2]
    
    P -->|Replicates| F1
    P -->|Replicates| F2
    
    style P fill:#4caf50,color:#fff
    style F1 fill:#2196f3,color:#fff
    style F2 fill:#2196f3,color:#fff
```

**Also known as**: Leader-Follower, Master-Slave, Single-Master

---

## 🔄 Replication Modes

### Synchronous Replication

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Primary
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    C->>P: INSERT user 'Alice'
    P->>P: Write locally
    P->>F1: Replicate
    P->>F2: Replicate
    F1->>P: ACK ✅
    F2->>P: ACK ✅
    P->>C: Commit Success
```

**Properties**:
- ✅ Strong consistency (all replicas in sync)
- ✅ No data loss on primary failure
- ❌ Higher latency (wait for all ACKs)
- ❌ Availability drops if any replica is slow/down

### Asynchronous Replication

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Primary
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    C->>P: INSERT user 'Alice'
    P->>P: Write locally
    P->>C: Commit Success
    
    Note over C: Client continues
    
    P-->>F1: Replicate (background)
    P-->>F2: Replicate (background)
```

**Properties**:
- ✅ Lower latency
- ✅ Higher availability
- ❌ Followers may lag (replication lag)
- ❌ Data loss possible if primary fails

---

## 📊 Replication Lag

```mermaid
graph LR
    subgraph "Replication Lag Scenario"
        P[Primary<br/>X=5]
        F1[Follower 1<br/>X=5]
        F2[Follower 2<br/>X=3 ⚠️]
    end
    
    Note[Follower 2 is behind!<br/>Reading from it returns stale data]
```

**Causes**:
- Network latency
- Follower overload
- Long-running queries on followers
- Large transactions

**Real-World Issue**: User writes, then reads from follower → doesn't see their own write!

---

## 🔄 Failover

When the primary fails, a follower must take over.

### Manual Failover

```mermaid
sequenceDiagram
    participant Op as Operator
    participant P as Primary
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    Note over P: ❌ CRASH
    Op->>Op: Detect failure
    Op->>F1: Promote to primary
    Op->>F2: Point to new primary
    F1->>F1: Now accepting writes
```

**Pros**: Safe, controlled  
**Cons**: Downtime during manual intervention

### Automated Failover

```mermaid
sequenceDiagram
    participant P as Primary
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    Note over P: ❌ CRASH
    F1->>P: Heartbeat timeout
    F2->>P: Heartbeat timeout
    F1->>F2: Election: I have latest data
    F2->>F1: OK, you're the new primary
    F1->>F1: Promoted!
```

**Pros**: Fast recovery  
**Cons**: Risk of split-brain, data loss

---

## ⚠️ Failover Challenges

### 1. Split-Brain

```mermaid
graph TB
    subgraph "Network Partition"
        P1[Old Primary<br/>Still thinks it's leader]
        P2[New Primary<br/>Promoted by followers]
    end
    
    C1[Client 1] -->|Writes| P1
    C2[Client 2] -->|Writes| P2
    
    Conflict((Both accepting writes!<br/>DATA CONFLICT))
    
    style Conflict fill:#f44336,color:#fff
```

**Solution**: Fencing tokens, STONITH (Shoot The Other Node In The Head)

### 2. Data Loss with Async Replication

```mermaid
graph LR
    P[Old Primary<br/>Writes: A, B, C] -->|Crash| X((Lost C))
    F[New Primary<br/>Writes: A, B] -->|Promoted| New[New Primary]
    
    style X fill:#f44336,color:#fff
```

If primary crashes before replicating recent writes → those writes are lost.

---

## 🔥 Real-World: GitHub 2018 Outage (Detailed)

```mermaid
graph TB
    subgraph "What Happened"
        1[1. Network blip]
        2[2. Orchestrator promotes replica]
        3[3. Topology update fails]
        4[4. Old primary comes back]
        5[5. TWO PRIMARIES!]
        6[6. 24h to recover]
    end
    
    1 --> 2 --> 3 --> 4 --> 5 --> 6
```

**Root Cause**: 
- Orchestrator (automated failover tool) promoted a read replica
- Original primary came back online
- Both accepting writes → data divergence

**Lessons**:
1. Test failover procedures regularly
2. Have clear fencing mechanisms
3. Understand your replication lag SLA

---

## 🏢 Real Systems

| System | Replication Type | Failover |
|--------|------------------|----------|
| PostgreSQL | Streaming (sync/async) | Manual or Patroni |
| MySQL | Binlog (async default) | MHA, Orchestrator |
| MongoDB | OpLog (sync or async) | Automatic with replica sets |
| Redis | Async by default | Sentinel for auto-failover |

---

## ✅ Key Takeaways

1. **Single leader** handles all writes → serializes operations
2. **Sync replication** = no data loss but higher latency
3. **Async replication** = lower latency but possible data loss
4. **Replication lag** can cause stale reads
5. **Failover** is complex — split-brain is a real risk
6. **Most databases** use this pattern (PostgreSQL, MySQL, MongoDB)

---

[← Previous: Replication Fundamentals](./03-replication-fundamentals.md) | [Next: Multi-Primary Replication →](./05-multi-primary-replication.md)
