# System Models

> Defining properties that distributed systems must satisfy to reason about correctness.

---

## 🎯 Why System Models Matter

Real-world distributed systems vary dramatically. To solve problems generically, we need **abstract models** that define the properties a system must satisfy.

```mermaid
graph TB
    subgraph "Real Systems"
        R1[Internet Apps]
        R2[Data Centers]
        R3[IoT Networks]
    end
    
    subgraph "Abstract Model"
        M[Common Properties]
    end
    
    subgraph "Solutions"
        S1[Algorithms]
        S2[Protocols]
    end
    
    R1 --> M
    R2 --> M
    R3 --> M
    M --> S1
    M --> S2
```

---

## ⏱️ Timing Models

### 1. Synchronous System

All nodes have accurate clocks and known bounds on message delay.

```mermaid
sequenceDiagram
    participant A as Node A
    participant B as Node B
    
    Note over A,B: Known max delay: 100ms
    A->>B: Message (50ms)
    Note over A,B: ✅ Within bounds
    B->>A: Response (30ms)
    Note over A: Total time predictable
```

**Properties**:
- ✅ Known upper bound on message delay
- ✅ Known upper bound on processing time
- ✅ Accurate synchronized clocks
- ✅ Execution in lock-step rounds

**Examples**: Tightly coupled clusters, real-time systems

---

### 2. Asynchronous System

No timing guarantees whatsoever. Messages can take arbitrarily long.

```mermaid
sequenceDiagram
    participant A as Node A
    participant B as Node B
    
    Note over A,B: No timing guarantees
    A->>B: Message
    Note over B: Processing...<br/>(unknown time)
    B-->>A: Response<br/>(eventually)
    Note over A: When? 🤷
```

**Properties**:
- ❌ No bound on message delay
- ❌ No bound on processing time
- ❌ No synchronized clocks
- ❌ Nodes run at independent rates

**Examples**: The Internet, most cloud systems

---

### Comparison

| Property | Synchronous | Asynchronous |
|----------|-------------|--------------|
| Message delay | Bounded | Unbounded |
| Clock sync | Yes | No |
| Easier to reason about | ✅ Yes | ❌ No |
| Realistic for Internet | ❌ No | ✅ Yes |

> **Most algorithms in this course assume the asynchronous model** because it's closer to real-world systems like the Internet.

---

## 🌐 Network Models

### Reliable Network
- Messages are eventually delivered
- No message loss
- **Rare in practice**

### Fair-Loss Network
- Messages may be lost
- If you keep retrying, message eventually gets through
- **More realistic**

### Arbitrary/Byzantine Network
- Messages can be lost, duplicated, or corrupted
- Network may behave maliciously
- **Most pessimistic**

```mermaid
graph LR
    subgraph "Reliable"
        R1[A] -->|Always delivered| R2[B]
    end
    
    subgraph "Fair-Loss"
        F1[A] -->|May need retries| F2[B]
    end
    
    subgraph "Byzantine"
        B1[A] -->|?? Corrupted ??| B2[B]
    end
```

---

## 🔧 Challenges in Asynchronous Systems

### Network Asynchrony
Messages can be delayed indefinitely, making it hard to distinguish between:
- A slow node
- A dead node
- A network partition

```mermaid
graph TB
    A[Node A] -->|Ping| B[Node B]
    A -->|No response...| A
    
    Q1{Is Node B dead?}
    Q2{Is Node B slow?}
    Q3{Is network broken?}
    
    A --> Q1
    A --> Q2
    A --> Q3
    
    style Q1 fill:#ffcdd2
    style Q2 fill:#fff9c4
    style Q3 fill:#c8e6c9
```

### Partial Failures
Only some components fail while others continue operating.

### Concurrency
Multiple operations happen simultaneously with no global ordering.

---

## 🏢 Real-World: Google Spanner

Google Spanner bridges synchronous and asynchronous models:

```mermaid
graph TB
    subgraph "Spanner Architecture"
        GPS[GPS Receivers]
        Atomic[Atomic Clocks]
        TT[TrueTime API]
        
        GPS --> TT
        Atomic --> TT
        
        TT --> Node1[Spanner Node 1]
        TT --> Node2[Spanner Node 2]
        TT --> Node3[Spanner Node 3]
    end
```

- Uses **GPS + atomic clocks** to achieve tight clock synchronization
- **TrueTime API** returns a time interval, not a single point
- Allows **external consistency** (stronger than linearizability)

---

## ✅ Key Takeaways

1. **Synchronous systems** have timing guarantees — easier to reason about but unrealistic for Internet
2. **Asynchronous systems** have no timing guarantees — realistic but harder to build
3. Most distributed systems algorithms assume **asynchronous model**
4. In async systems, **impossible to distinguish slow nodes from dead ones**
5. **Timeouts** are artificial bounds we impose, not actual limits

---

[← Previous: Fallacies](./02-fallacies-of-distributed-computing.md) | [Next: Types of Failures →](./04-types-of-failures.md)
