# Paxos Consensus Algorithm

> The foundational consensus algorithm — complex but correct.

---

## 🎯 Overview

Paxos solves consensus in an asynchronous system that may have failures.

```mermaid
graph TB
    subgraph "Paxos Roles"
        P[Proposers<br/>Propose values]
        A[Acceptors<br/>Vote on proposals]
        L[Learners<br/>Learn decided value]
    end
    
    P --> A
    A --> L
```

**One node can play multiple roles.**

---

## 📋 The Two Phases

### Phase 1: Prepare

```mermaid
sequenceDiagram
    participant P as Proposer
    participant A1 as Acceptor 1
    participant A2 as Acceptor 2
    participant A3 as Acceptor 3
    
    Note over P: Pick proposal number n=5
    P->>A1: Prepare(5)
    P->>A2: Prepare(5)
    P->>A3: Prepare(5)
    
    A1->>P: Promise(5, no prior)
    A2->>P: Promise(5, no prior)
    A3->>P: Promise(5, no prior)
    
    Note over P: Majority promised!
```

**Prepare message**: "I want to make proposal #n"  
**Promise response**: "I won't accept proposals < n" (+ any value already accepted)

### Phase 2: Accept

```mermaid
sequenceDiagram
    participant P as Proposer
    participant A1 as Acceptor 1
    participant A2 as Acceptor 2
    participant A3 as Acceptor 3
    
    Note over P: Majority promised, send Accept
    P->>A1: Accept(5, value="X")
    P->>A2: Accept(5, value="X")
    P->>A3: Accept(5, value="X")
    
    A1->>P: Accepted(5)
    A2->>P: Accepted(5)
    A3->>P: Accepted(5)
    
    Note over P: Consensus reached: X
```

---

## 🔒 Safety Rules

### Acceptor Rules

1. **On Prepare(n)**: If n > highest seen, promise to ignore proposals < n
2. **On Accept(n, v)**: Accept if n >= highest promised

### Proposer Rules

1. Pick unique, increasing proposal numbers
2. If any acceptor already accepted a value, must use that value

---

## 🔄 Handling Conflicts

```mermaid
sequenceDiagram
    participant P1 as Proposer 1
    participant P2 as Proposer 2
    participant A as Acceptors
    
    P1->>A: Prepare(1)
    A->>P1: Promise(1)
    
    P2->>A: Prepare(2)
    A->>P2: Promise(2)
    
    P1->>A: Accept(1, "X")
    A--xP1: Rejected! (promised 2)
    
    P2->>A: Accept(2, "Y")
    A->>P2: Accepted!
```

**Higher proposal number wins**, but existing values are preserved.

---

## 🔧 Multi-Paxos

Basic Paxos decides **one value**. Multi-Paxos extends it for a log:

```mermaid
graph TB
    subgraph "Multi-Paxos Log"
        S1[Slot 1: Paxos → A]
        S2[Slot 2: Paxos → B]
        S3[Slot 3: Paxos → C]
        S4[Slot 4: ...]
    end
    
    Leader[Stable Leader<br/>Skip Phase 1!]
    Leader --> S1
    Leader --> S2
    Leader --> S3
```

**Optimization**: With a stable leader, skip Phase 1 for subsequent slots.

---

## 📊 Paxos Variants

| Variant | Improvement |
|---------|-------------|
| **Multi-Paxos** | Log of decisions, leader optimization |
| **Fast Paxos** | Fewer round trips in common case |
| **Cheap Paxos** | Fewer acceptors in normal case |
| **Egalitarian Paxos** | No designated leader |

---

## ⚠️ Why Paxos Is Hard

1. **Liveness not guaranteed**: Dueling proposers can prevent progress
2. **Complex edge cases**: Recovery, reconfiguration
3. **Underspecified**: Original paper leaves implementation details vague
4. **Hard to understand**: Lamport himself wrote "Paxos Made Simple" later

---

## 🏢 Real-World: Google Chubby

```mermaid
graph TB
    subgraph "Google Chubby"
        Paxos[Paxos for<br/>leader election]
        Lock[Distributed<br/>lock service]
        GFS[Used by GFS,<br/>BigTable, etc.]
    end
    
    Paxos --> Lock
    Lock --> GFS
```

**Chubby uses Paxos** for:
- Leader election
- Distributed locking
- Storing small amounts of metadata

---

## 📋 Paxos vs Raft

| Aspect | Paxos | Raft |
|--------|-------|------|
| Understandability | ❌ Complex | ✅ Designed for clarity |
| Leader | Optional | Required |
| Log structure | Implicit | Explicit |
| Practical implementations | Harder | Easier |

---

## ✅ Key Takeaways

1. **Paxos** = Classic consensus, two phases (Prepare, Accept)
2. **Majority quorum** required for each phase
3. **Safety always maintained**, liveness not guaranteed
4. **Multi-Paxos** for log of decisions with leader optimization
5. **Hard to implement** — Raft is often preferred today
6. **Used by**: Google (Chubby, Spanner), Amazon (DynamoDB internals)

---

[← Previous: FLP Impossibility](./02-flp-impossibility.md) | [Next: Raft →](./04-raft.md)
