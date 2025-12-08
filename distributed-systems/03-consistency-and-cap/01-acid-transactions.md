# ACID Transactions

> The bedrock of database reliability — but what does it really mean in distributed systems?

---

## 🎯 The ACID Properties

```mermaid
graph TB
    subgraph "ACID"
        A[Atomicity<br/>All or nothing]
        C[Consistency<br/>Valid state transitions]
        I[Isolation<br/>Concurrent txns don't interfere]
        D[Durability<br/>Committed = permanent]
    end
```

---

## ⚛️ Atomicity

> **"All or nothing"** — either all operations in a transaction succeed, or none do.

```mermaid
sequenceDiagram
    participant C as Client
    participant DB as Database
    
    C->>DB: BEGIN TRANSACTION
    C->>DB: Deduct $100 from Account A
    C->>DB: Add $100 to Account B
    
    alt Success
        C->>DB: COMMIT
        DB->>C: ✅ Both operations saved
    else Failure (e.g., Account B not found)
        C->>DB: ROLLBACK
        DB->>C: ❌ Neither operation saved
    end
```

### In Distributed Systems

```mermaid
graph TB
    subgraph "Distributed Atomicity Challenge"
        TM[Transaction Manager]
        N1[Node 1: Deduct $100]
        N2[Node 2: Add $100]
        
        TM --> N1
        TM --> N2
        
        Q{What if N2 fails<br/>after N1 succeeds?}
    end
    
    style Q fill:#fff9c4
```

**Solution**: 2-Phase Commit (2PC), Saga Pattern — covered in Module 4

---

## 🔄 Consistency (ACID version)

> Transactions take the database from one **valid state** to another **valid state**.

⚠️ **NOT the same as CAP consistency!**

```mermaid
graph LR
    subgraph "Valid States"
        S1[State 1<br/>Total balance: $1000]
        S2[State 2<br/>Total balance: $1000]
    end
    
    S1 -->|Transfer $100| S2
    
    Invalid[Invalid State<br/>Total: $900 ⚠️]
    
    style Invalid fill:#f44336,color:#fff
```

**Examples of constraints**:
- Foreign key relationships
- Unique constraints
- Check constraints (balance >= 0)
- Application-level invariants

### ACID Consistency vs CAP Consistency

| ACID Consistency | CAP Consistency |
|------------------|-----------------|
| Valid state after transaction | All nodes see same data |
| Application-defined rules | Every read gets latest write |
| Enforced by database | Replicas in sync |
| Single-node concept | Distributed concept |

---

## 🔒 Isolation

> Concurrent transactions don't see each other's intermediate states.

```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant DB as Database
    participant T2 as Transaction 2
    
    T1->>DB: Read balance: $100
    T2->>DB: Read balance: $100
    T1->>DB: balance = $100 + $50
    T2->>DB: balance = $100 + $30
    T1->>DB: COMMIT (balance = $150)
    T2->>DB: COMMIT (balance = $130)
    
    Note over DB: Final: $130 ❌<br/>Should be: $180
```

**Lost Update Problem!** — we'll cover isolation levels in detail later.

---

## 💾 Durability

> Once committed, data survives crashes, power outages, etc.

```mermaid
graph TB
    subgraph "Durability Implementation"
        WAL[Write-Ahead Log]
        Disk[Persistent Storage]
        Rep[Replication]
    end
    
    TX[Transaction] --> WAL
    WAL --> Disk
    WAL --> Rep
    
    Note[If crash, replay from log]
```

### In Distributed Systems

Single-node durability (writing to disk) isn't enough:
- Disk could fail
- Entire datacenter could go down

**Distributed durability**: Replicate to multiple nodes in different locations.

---

## 🏢 Real-World: How Databases Implement ACID

### PostgreSQL

```mermaid
graph TB
    subgraph "PostgreSQL ACID"
        WAL[WAL - Write Ahead Log<br/>Atomicity + Durability]
        MVCC[MVCC<br/>Isolation]
        Constraints[Constraints<br/>Consistency]
    end
```

### MongoDB (with Transactions)

```javascript
// MongoDB transaction example
session.startTransaction();
try {
    await coll.updateOne({ _id: 1 }, { $inc: { balance: -100 } }, { session });
    await coll.updateOne({ _id: 2 }, { $inc: { balance: 100 } }, { session });
    await session.commitTransaction();
} catch (error) {
    await session.abortTransaction();
}
```

---

## ⚠️ ACID in Distributed Systems: The Challenge

```mermaid
graph TB
    subgraph "Single-Node Database"
        Single[All ACID properties achievable ✅]
    end
    
    subgraph "Distributed Database"
        D1[Atomicity - Need 2PC/Saga]
        D2[Consistency - App-defined]
        D3[Isolation - Complex across shards]
        D4[Durability - Replication]
    end
    
    Note[Much harder to achieve<br/>all four properties!]
```

Many distributed databases provide:
- **Weaker isolation levels** (eventual consistency)
- **Single-row ACID only** (no multi-row transactions)
- **Opt-in strong consistency** (at performance cost)

---

## 📊 ACID Support by Database

| Database | Atomicity | Consistency | Isolation | Durability |
|----------|-----------|-------------|-----------|------------|
| PostgreSQL | ✅ Full | ✅ Constraints | ✅ Serializable | ✅ WAL |
| MySQL | ✅ Full | ✅ Constraints | ⚠️ Default: Repeatable Read | ✅ InnoDB |
| MongoDB | ✅ Multi-doc (4.0+) | ⚠️ App-level | ⚠️ Snapshot | ✅ WiredTiger |
| Cassandra | ⚠️ Row-level | ❌ Limited | ⚠️ Limited | ✅ Commitlog |
| DynamoDB | ⚠️ Item-level | ❌ Limited | ⚠️ Eventual/Strong | ✅ Multi-AZ |

---

## ✅ Key Takeaways

1. **Atomicity**: All or nothing — harder across nodes, need 2PC or Saga
2. **Consistency (ACID)**: Valid state transitions — NOT the same as CAP
3. **Isolation**: Concurrent transactions appear sequential — many levels exist
4. **Durability**: Committed = permanent — replication for distributed durability
5. **Full ACID in distributed systems is expensive** — often trade-offs are made

---

[← Back to Module](./README.md) | [Next: CAP Theorem →](./02-cap-theorem.md)
