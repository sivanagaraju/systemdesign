# The PACELC Theorem

> CAP's practical extension — what happens when there's NO partition?

---

## 🎯 The Problem with CAP

CAP only tells us what happens **during** a partition. But partitions are rare!

**What about normal operation?**

```mermaid
graph LR
    CAP[CAP] --> Q{During partition?}
    Q -->|Yes| Choice[Choose C or A]
    Q -->|No| Missing[??? No guidance]
    
    style Missing fill:#fff9c4
```

---

## 💡 PACELC: The Full Picture

> **P**artition → Choose **A**vailability or **C**onsistency  
> **E**lse (no partition) → Choose **L**atency or **C**onsistency

```mermaid
graph TB
    subgraph "PACELC"
        P[Partition?]
        
        P -->|Yes| PA_PC[PA or PC?]
        P -->|No| EL_EC[EL or EC?]
        
        PA_PC --> PA[PA: Prioritize Availability]
        PA_PC --> PC[PC: Prioritize Consistency]
        
        EL_EC --> EL[EL: Prioritize Latency]
        EL_EC --> EC[EC: Prioritize Consistency]
    end
```

---

## 📊 Why Latency Matters

Even without partitions, there's a trade-off:

```mermaid
sequenceDiagram
    participant C as Client
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3
    
    alt High Consistency (EC)
        C->>N1: Write X=5
        N1->>N2: Sync replicate
        N1->>N3: Sync replicate
        N2->>N1: ACK
        N3->>N1: ACK
        N1->>C: Success (slower)
    else Low Latency (EL)
        C->>N1: Write X=5
        N1->>C: Success (faster)
        N1-->>N2: Async replicate
        N1-->>N3: Async replicate
    end
```

**EC (Else Consistency)**: Wait for replication → Higher latency  
**EL (Else Latency)**: Return immediately → Lower latency, eventual consistency

---

## 🗂️ PACELC Classification

```mermaid
graph TB
    subgraph "PA/EL Systems"
        PA_EL[Prioritize Availability & Latency]
        PAEL1[Cassandra default]
        PAEL2[DynamoDB default]
        PAEL3[Riak]
    end
    
    subgraph "PC/EC Systems"
        PC_EC[Prioritize Consistency Always]
        PCEC1[Google Spanner]
        PCEC2[VoltDB]
        PCEC3[HBase]
    end
    
    subgraph "PA/EC Systems"
        PA_EC[Available in partition,<br/>Consistent otherwise]
        PAEC1[MongoDB default]
        PAEC2[Some configs]
    end
```

---

## 📋 System Classification Table

| System | P (Partition) | E (Else/Normal) | Notes |
|--------|---------------|-----------------|-------|
| **Cassandra** | PA | EL | Tunable consistency per query |
| **DynamoDB** | PA | EL | Default eventual, optional strong |
| **Riak** | PA | EL | Designed for availability |
| **Spanner** | PC | EC | TrueTime enables strong consistency |
| **ZooKeeper** | PC | EC | Coordination requires consistency |
| **MongoDB** | PA | EC | Default: available, then consistent |
| **CockroachDB** | PC | EC | Serializable by default |
| **MySQL (async)** | PA | EL | Async replication for performance |

---

## 🔧 Tunable Consistency

Many modern databases let you choose per-operation:

```mermaid
graph TB
    subgraph "Cassandra Consistency Levels"
        Op[Operation: Read/Write]
        
        Op --> ONE[ONE<br/>Fast, eventual]
        Op --> QUORUM[QUORUM<br/>Balanced]
        Op --> ALL[ALL<br/>Strong, slow]
    end
```

### Cassandra Example

```cql
-- Fast write (PA/EL)
INSERT INTO users (id, name) VALUES (1, 'Alice')
  USING CONSISTENCY ONE;

-- Consistent write (PC/EC)
INSERT INTO users (id, name) VALUES (1, 'Alice')
  USING CONSISTENCY ALL;

-- Balanced read
SELECT * FROM users WHERE id = 1
  USING CONSISTENCY QUORUM;
```

---

## 🏢 Real-World Decisions

### Netflix: PA/EL by Default

```mermaid
graph TB
    subgraph "Netflix Stack"
        API[API Layer]
        Cass[(Cassandra<br/>PA/EL)]
        EVCache[(EVCache<br/>PA/EL)]
    end
    
    API --> Cass
    API --> EVCache
    
    Note[Availability > Consistency<br/>for viewing history, recommendations]
```

**Why?**: Better to show slightly stale recommendations than error.

### Stripe: PC/EC for Payments

```mermaid
graph TB
    subgraph "Stripe Stack"
        API[API Layer]
        DB[(PostgreSQL<br/>PC/EC)]
        Idempotency[Idempotency Keys]
    end
    
    API --> DB
    API --> Idempotency
    
    Note[Consistency critical<br/>for financial data]
```

**Why?**: Money must never be lost or duplicated.

---

## 📈 Making the Choice

```mermaid
graph TB
    Q1{Data type?}
    
    Q1 -->|Financial, Inventory| PC_EC[PC/EC: Strong consistency]
    Q1 -->|Social, Analytics| PA_EL[PA/EL: High availability]
    Q1 -->|Mixed| Tunable[Tunable: Per-operation]
    
    PC_EC --> Ex1[Spanner, CockroachDB]
    PA_EL --> Ex2[Cassandra, DynamoDB]
    Tunable --> Ex3[Cassandra, DynamoDB with configs]
```

---

## 📊 Quick Reference

| Category | During Partition | Normal Operation | Use Case |
|----------|------------------|------------------|----------|
| **PA/EL** | Available | Low latency | Social feeds, IoT |
| **PA/EC** | Available | Consistent | General apps |
| **PC/EL** | Consistent | Low latency | (Rare) |
| **PC/EC** | Consistent | Consistent | Banking, inventory |

---

## ✅ Key Takeaways

1. **PACELC extends CAP** with what happens during normal operation
2. **The tradeoff**: Latency vs Consistency (even without partitions)
3. **PA/EL systems** (Cassandra, DynamoDB): Fast but eventually consistent
4. **PC/EC systems** (Spanner, ZooKeeper): Consistent but higher latency
5. **Modern databases are tunable** — choose per operation
6. **Choose based on your data**: Financial = PC/EC, Social = PA/EL

---

[← Previous: CAP Theorem](./02-cap-theorem.md) | [Next: Consistency Models →](./04-consistency-models.md)
