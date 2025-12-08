# Physical vs Logical Time

> Why you can't trust wall clocks in distributed systems.

---

## 🎯 The Problem

```mermaid
graph TB
    subgraph "Real-World Scenario"
        N1[Node 1<br/>Clock: 10:00:00.000]
        N2[Node 2<br/>Clock: 10:00:00.050]
        N3[Node 3<br/>Clock: 09:59:59.980]
    end
    
    Note[Same moment in time<br/>but different readings!]
```

**Physical clocks are never perfectly synchronized.**

---

## 🕐 Physical Time Problems

### 1. Clock Drift

Clocks run at slightly different rates:
- Typical drift: 10-100 ppm (parts per million)
- 100 ppm = ~8 seconds per day

### 2. Clock Skew

Even after synchronization, clocks differ:
```
NTP accuracy: ~10ms over internet
GPS accuracy: ~100ns (expensive)
```

### 3. Clock Jumps

Clocks can jump forward or **backward**:
- NTP corrections
- Leap seconds
- VM migrations

```mermaid
sequenceDiagram
    participant C as Clock
    
    C->>C: 10:00:00
    C->>C: 10:00:01
    C->>C: 10:00:02
    Note over C: NTP correction!
    C->>C: 09:59:58 (jumped back!)
```

---

## 🔥 Real-World Incident: Cloudflare Leap Second

**What happened** (2017):
1. Leap second added at midnight
2. Some servers had time go backward
3. Software assumed time only goes forward
4. Negative durations caused crashes

**Lesson**: Never assume monotonic physical time!

---

## 💡 Logical Time

Instead of asking "What time is it?", ask "What happened before what?"

```mermaid
graph LR
    subgraph "Logical Time"
        E1[Event A<br/>Logical time: 1]
        E2[Event B<br/>Logical time: 2]
        E3[Event C<br/>Logical time: 3]
    end
    
    E1 --> E2 --> E3
```

**Logical clocks** capture ordering, not actual time.

---

## 📊 Comparison

| Aspect | Physical Time | Logical Time |
|--------|---------------|--------------|
| Source | Hardware clock | Counter |
| Accuracy | Approximate | Exact ordering |
| Synchronized? | Approximately | N/A |
| Jumps backward? | Yes | Never |
| Cross-system comparison | Unreliable | Reliable |

---

## 🔧 When to Use What

```mermaid
graph TB
    Q1{Need real<br/>wall clock time?}
    
    Q1 -->|Yes| Physical[Physical Time<br/>+ Bounded uncertainty]
    Q1 -->|No| Q2{Need causality?}
    
    Q2 -->|Yes| Vector[Vector Clocks]
    Q2 -->|No| Lamport[Lamport Clocks]
    
    Physical --> Spanner[Google Spanner]
    Vector --> Dynamo[DynamoDB, Riak]
    Lamport --> Raft[Raft, Paxos]
```

---

## 🏢 Google Spanner's TrueTime

Spanner uses **bounded physical time**:

```mermaid
graph TB
    subgraph "TrueTime API"
        GPS[GPS Receivers]
        Atomic[Atomic Clocks]
        
        GPS --> TT[TrueTime]
        Atomic --> TT
        
        TT --> Response[earliest, latest]
    end
    
    Note[Returns interval,<br/>not single point!]
```

```go
// TrueTime API
interval := TrueTime.Now()
// interval = [earliest, latest]
// Actual time is guaranteed to be within this range
```

---

## ✅ Key Takeaways

1. **Physical clocks drift, skew, and jump** — don't trust them for ordering
2. **Logical clocks** provide ordering without real time
3. **Lamport clocks**: Simple, total ordering
4. **Vector clocks**: Detect causality
5. **TrueTime**: Bounded uncertainty for physical time (expensive)
6. **Choose based on need**: Ordering only? Use logical. Need wall time? Add uncertainty bounds.

---

[← Back to Module](./README.md) | [Next: Lamport Clocks →](./02-lamport-clocks.md)
