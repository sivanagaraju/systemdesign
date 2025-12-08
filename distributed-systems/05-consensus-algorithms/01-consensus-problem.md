# The Consensus Problem

> Getting distributed nodes to agree on a single value.

---

## 🎯 What is Consensus?

> **Consensus** = All nodes agreeing on the same value, even when some nodes fail.

```mermaid
graph TB
    subgraph "Consensus Example"
        N1[Node 1<br/>Proposes: A]
        N2[Node 2<br/>Proposes: B]
        N3[Node 3<br/>Proposes: C]
        
        Result[All agree: B ✅]
        
        N1 --> Result
        N2 --> Result
        N3 --> Result
    end
```

---

## 📋 Requirements

A valid consensus algorithm must satisfy:

### 1. Agreement

All non-faulty nodes decide on the **same value**.

```mermaid
graph TB
    N1[Node 1: Decides X]
    N2[Node 2: Decides X]
    N3[Node 3: Decides X]
    
    style N1 fill:#4caf50,color:#fff
    style N2 fill:#4caf50,color:#fff
    style N3 fill:#4caf50,color:#fff
```

### 2. Validity

The decided value must have been proposed by some node.

```mermaid
graph TB
    Proposed[Proposed values: A, B, C]
    Decided[Decided: B ✅]
    Invalid[Decided: Z ❌<br/>Never proposed!]
    
    Proposed --> Decided
    Proposed -.->|Invalid| Invalid
    
    style Invalid fill:#f44336,color:#fff
```

### 3. Termination (Liveness)

All non-faulty nodes **eventually** decide.

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3
    
    Note over N1,N3: Process...
    Note over N1,N3: Eventually...
    N1->>N1: Decides!
    N2->>N2: Decides!
    N3->>N3: Decides!
```

---

## 🔧 Use Cases for Consensus

### 1. Leader Election

```mermaid
graph TB
    subgraph "Leader Election"
        C1[Candidate 1]
        C2[Candidate 2]
        C3[Candidate 3]
        
        Vote[Consensus:<br/>Who is leader?]
        
        C1 --> Vote
        C2 --> Vote
        C3 --> Vote
        
        Vote --> Leader[C2 is leader!]
    end
```

### 2. Distributed Locking

```mermaid
graph TB
    subgraph "Distributed Lock"
        P1[Process 1: wants lock]
        P2[Process 2: wants lock]
        
        Consensus[Consensus:<br/>Who gets lock?]
        
        P1 --> Consensus
        P2 --> Consensus
        
        Consensus --> Winner[P1 gets lock!]
    end
```

### 3. Atomic Broadcast / State Machine Replication

```mermaid
graph TB
    subgraph "Replicated Log"
        Log1[Node 1 Log: A, B, C]
        Log2[Node 2 Log: A, B, C]
        Log3[Node 3 Log: A, B, C]
        
        Note[All nodes have<br/>same log order!]
    end
```

---

## ⚠️ Why Is It Hard?

### 1. Failures Happen

```mermaid
graph TB
    N1[Node 1 ✅]
    N2[Node 2 ❌ Crashed]
    N3[Node 3 ✅]
    N4[Node 4 ⚠️ Slow]
    
    Q{How do remaining<br/>nodes agree?}
```

### 2. Network Delays

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3
    
    N1->>N2: Vote for A
    Note over N1,N3: Message delayed...
    N3->>N2: Vote for B
    
    Note over N2: Which arrived first?<br/>Depends on network!
```

### 3. Partial Failures

```mermaid
graph TB
    subgraph "Partial Failure"
        N1[N1 sees: N2 alive, N3 dead]
        N2[N2 sees: N1 alive, N3 alive]
        N3[N3 sees: N1 dead, N2 alive]
    end
    
    Q{Different nodes have<br/>different views!}
```

---

## 🔥 Real-World: Cloudflare 2020 Outage

**What happened**:
1. Multiple etcd nodes in cluster
2. Leader election triggered
3. Bug caused repeated elections
4. No stable leader → service degradation

```mermaid
graph TB
    subgraph "The Problem"
        E1[etcd 1] -->|I'm leader!| Conflict
        E2[etcd 2] -->|No, I'm leader!| Conflict
        E3[etcd 3] -->|No, I'm leader!| Conflict
        
        Conflict[Repeated elections<br/>No progress!]
    end
    
    style Conflict fill:#f44336,color:#fff
```

**Lesson**: Consensus algorithms need careful configuration and testing.

---

## 📊 Summary

| Requirement | Description | Challenge |
|-------------|-------------|-----------|
| Agreement | All decide same | Failures change views |
| Validity | Decided was proposed | Trivial |
| Termination | All eventually decide | FLP impossibility! |

---

## ✅ Key Takeaways

1. **Consensus** = agreeing on a single value despite failures
2. **Three requirements**: Agreement, Validity, Termination
3. **Use cases**: Leader election, distributed locks, state machine replication
4. **Hard because**: Failures, delays, partial failures
5. **FLP says**: Can't guarantee termination in async systems with failures

---

[← Back to Module](./README.md) | [Next: FLP Impossibility →](./02-flp-impossibility.md)
