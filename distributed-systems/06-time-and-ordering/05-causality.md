# Causality in Distributed Systems

> Understanding "what caused what" without global time.

---

## 📧 **Email Thread Analogy**

```
Alice → Bob: "Let's meet at 3pm"       [Email 1]
Bob → Alice: "Sure, see you then"       [Email 2]
Charlie → Alice: "Happy birthday!"      [Email 3]
```

- Email 2 was **caused by** Email 1 (reply)
- Email 3 is **concurrent** (independent)

This is **causality**!

---

## 🎯 Formal Definition

### Happened-Before Relation (→)

```mermaid
graph LR
    A[Event A] -->|→| B[Event B]
    Note["A happened-before B<br/>A could have caused B"]
```

**Three rules:**
1. Same process: If A comes before B, then A → B
2. Message: If A sends, B receives, then A → B
3. Transitive: If A → B and B → C, then A → C

---

## 🔗 Causality Chains

```mermaid
sequenceDiagram
    participant P1 as Process 1
    participant P2 as Process 2
    participant P3 as Process 3
    
    P1->>P1: A (write x)
    P1->>P2: send
    P2->>P2: B (read x)
    P2->>P3: send
    P3->>P3: C (use x)
    
    Note over P1,P3: A → B → C<br/>(causal chain)
```

---

## ⚡ Concurrent Events

If neither A → B nor B → A, then A and B are **concurrent** (A || B).

```mermaid
graph TB
    subgraph "Process 1"
        A[Event A]
    end
    
    subgraph "Process 2"
        B[Event B]
    end
    
    Note["No causal relationship<br/>A || B (concurrent)"]
```

---

## 🎭 **Dominoes Analogy**

```mermaid
graph LR
    subgraph "Causal"
        D1[Domino 1] -->|Knocks over| D2[Domino 2]
        D2 -->|Knocks over| D3[Domino 3]
    end
    
    subgraph "Concurrent"
        D4[Independent Domino]
    end
    
    Note["D4 fell independently<br/>Not caused by D1-D3"]
```

---

## 📊 Why Causality Matters

| Problem | Why Causality Helps |
|---------|---------------------|
| **Message ordering** | Show cause before effect |
| **Conflict detection** | Know if updates are related |
| **Debugging** | Trace what caused what |
| **Consistency** | Respect logical dependencies |

---

## 🔧 Tools for Tracking Causality

| Tool | What It Captures |
|------|------------------|
| Lamport Clock | Ordering (not full causality) |
| Vector Clock | Full causality ✅ |
| Version Vector | Per-replica causality |

---

## 🔥 Real-World: Social Media

```mermaid
graph TB
    Post["Alice posts: 'Hello!'"]
    Reply1["Bob replies: 'Hi!'"]
    Reply2["Carol replies: 'Hey!'"]
    Like["Dave likes the post"]
    
    Post --> Reply1
    Post --> Reply2
    Post --> Like
    
    Note["Post MUST show before replies<br/>Replies can appear in any order (concurrent)"]
```

**Causal consistency** ensures you never see a reply without the original post.

---

## 📋 Causality Violations

### The Problem

```mermaid
sequenceDiagram
    participant A as Alice
    participant B as Bob
    participant C as Carol
    
    A->>B: I got the job!
    B->>C: Congrats to Alice!
    
    Note over C: Carol sees "Congrats"<br/>but not the announcement ❌
```

### The Solution: Causal Ordering

Ensure messages are delivered respecting causality.

---

## ✅ Key Takeaways

1. **Causality** = "Could A have influenced B?"
2. **Happened-before** (→) captures potential causality
3. **Concurrent events** have no causal relationship
4. **Vector clocks** are the tool to track causality
5. **Causal consistency** is often sufficient (cheaper than strong)

| Remember | Analogy |
|----------|---------|
| Causal chain | Domino effect |
| Concurrent | Two people texting at same time |
| Happened-before | Email thread replies |

---

[← Previous: Version Vectors](./04-version-vectors.md) | [Back to Module →](./README.md)
