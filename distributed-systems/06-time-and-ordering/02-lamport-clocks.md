# Lamport Clocks

> The foundation of logical time in distributed systems.

---

## 🎯 The Idea

> "A happened before B" doesn't need wall clock time — just a logical counter.

```mermaid
graph LR
    A[Event A<br/>LC: 1] -->|Happened before| B[Event B<br/>LC: 2]
    B --> C[Event C<br/>LC: 3]
```

---

## 📋 The Algorithm

### Rules

1. **Before any event**: Increment local counter
2. **When sending message**: Attach current counter
3. **When receiving message**: `counter = max(local, received) + 1`

```mermaid
sequenceDiagram
    participant P1 as Process 1
    participant P2 as Process 2
    participant P3 as Process 3
    
    Note over P1: LC=1 (event)
    P1->>P2: Message (LC=1)
    Note over P2: LC=max(0,1)+1=2
    Note over P2: LC=3 (event)
    P2->>P3: Message (LC=3)
    Note over P3: LC=max(0,3)+1=4
    P2->>P1: Message (LC=3)
    Note over P1: LC=max(1,3)+1=4
```

---

## 🔧 Implementation

```python
class LamportClock:
    def __init__(self):
        self.counter = 0
    
    def local_event(self):
        self.counter += 1
        return self.counter
    
    def send(self):
        self.counter += 1
        return self.counter  # Attach to message
    
    def receive(self, received_counter):
        self.counter = max(self.counter, received_counter) + 1
        return self.counter
```

---

## ✅ What Lamport Clocks Guarantee

### If A happened-before B, then LC(A) < LC(B)

```mermaid
graph LR
    A[A: LC=2] -->|causes| B[B: LC=5]
    
    Note[LC-A < LC-B ✅]
```

### ⚠️ BUT: LC(A) < LC(B) does NOT mean A happened-before B

```mermaid
graph TB
    subgraph "Process 1"
        A[A: LC=2]
    end
    
    subgraph "Process 2"
        B[B: LC=3]
    end
    
    Note[A and B are concurrent!<br/>LC ordering doesn't imply causality]
```

---

## 📊 Limitations

| Guarantee | ✅ / ❌ |
|-----------|--------|
| Causality implies LC ordering | ✅ Yes |
| LC ordering implies causality | ❌ No |
| Detect concurrent events | ❌ No |
| Total ordering | ⚠️ With tie-breaker |

---

## 🔧 Creating Total Order

Lamport clocks + process ID = total order:

```python
def compare(event_a, event_b):
    if event_a.lc != event_b.lc:
        return event_a.lc - event_b.lc
    return event_a.process_id - event_b.process_id  # Tie-breaker
```

---

## 🏢 Real-World Usage

### Raft Terms

```mermaid
graph TB
    subgraph "Raft"
        T1[Term 1<br/>Leader A]
        T2[Term 2<br/>Leader B]
        T3[Term 3<br/>Leader A]
    end
    
    T1 --> T2 --> T3
    
    Note[Terms are Lamport clocks!]
```

### Paxos Proposal Numbers

Same concept — increasing numbers to order proposals.

---

## ✅ Key Takeaways

1. **Lamport clocks** = Simple counters incremented on events
2. **Causality → LC ordering** but NOT the reverse
3. **Cannot detect concurrency** — use vector clocks for that
4. **Used in**: Raft terms, Paxos proposals, database versioning
5. **Simple and efficient** — just one integer per event

---

[← Previous: Physical vs Logical](./01-physical-vs-logical-time.md) | [Next: Vector Clocks →](./03-vector-clocks.md)
