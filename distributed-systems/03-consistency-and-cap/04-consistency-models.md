# Consistency Models

> Different levels of consistency guarantees in distributed systems.

---

## 🎯 The Spectrum

```mermaid
graph LR
    Strong[Strong<br/>Linearizability] --> Seq[Sequential<br/>Consistency]
    Seq --> Causal[Causal<br/>Consistency]
    Causal --> Eventual[Eventual<br/>Consistency]
    
    subgraph "Trade-off"
        Stronger[Stronger guarantees<br/>Higher latency]
        Weaker[Weaker guarantees<br/>Lower latency]
    end
    
    Strong -.-> Stronger
    Eventual -.-> Weaker
```

---

## 1️⃣ Linearizability (Strongest)

> Operations appear to happen **instantaneously** at some point between call and return.

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant S as System
    participant C2 as Client 2
    
    Note over S: X = 0
    C1->>S: Write X=1 (starts)
    Note over S: X = 1 (committed)
    C1->>S: (returns)
    C2->>S: Read X
    S->>C2: X=1 ✅
    
    Note over C1,C2: Once write returns,<br/>all reads see it
```

**Also known as**: Strong consistency, Atomic consistency

### Properties
- Reads always return latest write
- Operations appear atomic
- **Real-time ordering** maintained

### Real-World Examples
- **Spanner**: Uses TrueTime for linearizability
- **ZooKeeper**: Znodes are linearizable
- **etcd**: Raft-based linearizability

### Cost
- Requires coordination (latency)
- Lower availability during partitions

---

## 2️⃣ Sequential Consistency

> Operations appear in **same order** to all clients, but not necessarily real-time order.

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant C2 as Client 2
    
    C1->>C1: Write A=1
    C2->>C2: Write B=2
    C1->>C1: Read B=2
    C2->>C2: Read A=1
    
    Note over C1,C2: Both see same order:<br/>A=1 before B=2<br/>(or vice versa)
```

### Key Difference from Linearizability

```mermaid
graph TB
    subgraph "Linearizable"
        L1[If write returns before read starts...]
        L2[...read MUST see the write]
    end
    
    subgraph "Sequential"
        S1[All clients see same order...]
        S2[...but not necessarily real-time order]
    end
```

---

## 3️⃣ Causal Consistency

> Causally related operations appear in correct order; concurrent operations may vary.

```mermaid
sequenceDiagram
    participant A as Alice
    participant B as Bob
    participant C as Carol
    
    A->>A: Post: "Hello!"
    B->>B: Sees "Hello!", Replies: "Hi Alice!"
    
    Note over A,C: Carol must see "Hello!"<br/>before "Hi Alice!"<br/>(causally related)
    
    C->>C: Posts: "Nice weather"
    
    Note over A,C: "Nice weather" can appear<br/>anywhere (concurrent)
```

### Causal Relationships

```mermaid
graph TB
    subgraph "Causally Related"
        W1[Write A] --> R1[Read A]
        R1 --> W2[Write B based on A]
    end
    
    subgraph "Concurrent"
        C1[Write X]
        C2[Write Y]
    end
```

**Rule**: If operation B "knows about" operation A, A must come before B.

---

## 4️⃣ Eventual Consistency (Weakest)

> If no new writes, all replicas **eventually** converge to same value.

```mermaid
sequenceDiagram
    participant C as Client
    participant R1 as Replica 1
    participant R2 as Replica 2
    
    C->>R1: Write X=5
    R1->>C: OK
    
    Note over R1,R2: R2 still has X=0
    
    C->>R2: Read X
    R2->>C: X=0 (stale!)
    
    Note over R1,R2: Eventually syncs...
    
    R1-->>R2: Sync X=5
    
    C->>R2: Read X
    R2->>C: X=5 ✅
```

### Variations

| Variant | Guarantee |
|---------|-----------|
| **Read-your-writes** | You see your own writes |
| **Monotonic reads** | Never see older values after newer |
| **Monotonic writes** | Your writes applied in order |
| **Session consistency** | Guarantees within a session |

---

## 📊 Comparison Table

| Model | Guarantee | Latency | Use Case |
|-------|-----------|---------|----------|
| **Linearizable** | Real-time, atomic | Highest | Leader election, locks |
| **Sequential** | Same order for all | High | Some consistency needed |
| **Causal** | Cause before effect | Medium | Social features |
| **Eventual** | Will converge | Lowest | Analytics, caching |

---

## 🏢 Real-World: Social Media Example

```mermaid
graph TB
    subgraph "Facebook Posts"
        Post[Post created]
        Like[Like added]
        Comment[Comment added]
        
        Post --> Like
        Post --> Comment
        Like -.->|Concurrent| Comment
    end
    
    Note[Post must show before<br/>likes and comments<br/>Likes/comments order can vary]
```

**Causal consistency is perfect here**:
- Post appears before reactions (causal)
- Order of reactions can vary (concurrent, OK)
- Lower latency than linearizable

---

## 🔥 Real-World Incident: Amazon S3 (2006)

**S3's eventual consistency caused issues**:

```mermaid
sequenceDiagram
    participant App as Application
    participant S3 as S3
    
    App->>S3: PUT object/config.json
    S3->>App: 200 OK
    App->>S3: GET object/config.json
    S3->>App: 404 Not Found ⚠️
    
    Note over App,S3: Eventual consistency!<br/>Object not yet visible
```

**Fix (2020)**: S3 now offers strong read-after-write consistency!

---

## 🔧 Choosing the Right Model

```mermaid
graph TB
    Q1{Need real-time<br/>accuracy?}
    
    Q1 -->|Yes| Lin[Linearizable]
    Q1 -->|No| Q2{Need order<br/>between related ops?}
    
    Q2 -->|Yes| Caus[Causal]
    Q2 -->|No| Q3{Can tolerate<br/>stale reads?}
    
    Q3 -->|Yes| Event[Eventual]
    Q3 -->|Some| Session[Session consistency]
```

---

## ✅ Key Takeaways

1. **Linearizability**: Strongest, most expensive — for coordination
2. **Sequential**: Same order for all, not real-time
3. **Causal**: Preserves cause-effect — good for social features
4. **Eventual**: Weakest, fastest — for analytics, caching
5. **Choose based on requirements** — don't over-engineer
6. **S3 incident shows** even big systems struggle with consistency

---

[← Previous: PACELC](./03-pacelc-theorem.md) | [Next: Isolation Levels →](./05-isolation-levels.md)
