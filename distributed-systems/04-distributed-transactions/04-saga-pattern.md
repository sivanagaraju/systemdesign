# The Saga Pattern

> Managing long-running distributed transactions through compensating actions.

---

## 🎯 The Problem with 2PC

```mermaid
graph TB
    2PC[2PC]
    
    2PC --> Lock[Holds locks<br/>entire duration]
    2PC --> Block[Blocks on failure]
    2PC --> Scale[Doesn't scale well]
    
    style Lock fill:#ffcdd2
    style Block fill:#ffcdd2
    style Scale fill:#ffcdd2
```

**For long-running transactions, 2PC is impractical!**

---

## 💡 The Saga Solution

Instead of one atomic transaction, break it into:
- **Local transactions** (each atomic by itself)
- **Compensating transactions** (to undo if needed)

```mermaid
graph LR
    T1[T1: Create Order] --> T2[T2: Reserve Stock]
    T2 --> T3[T3: Charge Payment]
    T3 --> T4[T4: Ship Order]
    
    T4 -.->|If fails| C3[C3: Refund]
    C3 -.-> C2[C2: Release Stock]
    C2 -.-> C1[C1: Cancel Order]
```

---

## 📋 Saga Types

### 1. Choreography (Event-Driven)

```mermaid
graph LR
    subgraph "Choreography"
        OS[Order Service] -->|OrderCreated| IS[Inventory Service]
        IS -->|StockReserved| PS[Payment Service]
        PS -->|PaymentProcessed| SS[Shipping Service]
    end
```

Each service:
1. Listens for events
2. Does its work
3. Publishes next event

**Pros**: Decoupled, simple  
**Cons**: Hard to track, scattered logic

### 2. Orchestration (Central Coordinator)

```mermaid
graph TB
    subgraph "Orchestration"
        O[Saga Orchestrator]
        
        O -->|Create order| OS[Order Service]
        O -->|Reserve stock| IS[Inventory Service]
        O -->|Process payment| PS[Payment Service]
        O -->|Ship order| SS[Shipping Service]
    end
```

Orchestrator:
1. Executes steps in order
2. Handles failures
3. Triggers compensations

**Pros**: Centralized logic, easier to understand  
**Cons**: Orchestrator is a coupling point

---

## 🔄 Compensation Example

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant Order as Orders
    participant Stock as Inventory
    participant Pay as Payment
    
    O->>Order: Create order
    Order->>O: ✅ Created
    
    O->>Stock: Reserve stock
    Stock->>O: ✅ Reserved
    
    O->>Pay: Charge card
    Pay->>O: ❌ FAILED
    
    Note over O: Start compensation!
    
    O->>Stock: Release stock (compensate)
    Stock->>O: ✅ Released
    
    O->>Order: Cancel order (compensate)
    Order->>O: ✅ Cancelled
```

---

## 🔧 Saga State Machine

```mermaid
stateDiagram-v2
    [*] --> OrderCreated
    OrderCreated --> StockReserved: Reserve OK
    OrderCreated --> OrderCancelled: Reserve FAIL
    StockReserved --> PaymentProcessed: Payment OK
    StockReserved --> StockReleased: Payment FAIL
    StockReleased --> OrderCancelled
    PaymentProcessed --> Shipped: Ship OK
    PaymentProcessed --> PaymentRefunded: Ship FAIL
    PaymentRefunded --> StockReleased
    Shipped --> [*]
    OrderCancelled --> [*]
```

---

## 🔥 Real-World: Uber Trip Saga

```mermaid
graph TB
    subgraph "Uber Trip Booking"
        T1[Create Trip]
        T2[Find Driver]
        T3[Reserve Driver]
        T4[Calculate Fare]
        T5[Charge Payment]
        
        T1 --> T2 --> T3 --> T4 --> T5
        
        T5 -.->|Fail| C4[Cancel Trip]
        C4 -.-> C3[Release Driver]
    end
```

**Uber uses Cadence/Temporal** for saga orchestration!

---

## 📊 Saga vs 2PC

| Aspect | 2PC | Saga |
|--------|-----|------|
| Atomicity | ✅ Full | ⚠️ Eventual (semantic) |
| Isolation | ✅ Full | ❌ None (intermediate visible) |
| Availability | ❌ Lower (blocking) | ✅ Higher |
| Latency | ❌ Higher (locks) | ✅ Lower |
| Complexity | Lower | Higher (compensations) |

---

## ⚠️ Saga Challenges

### 1. Dirty Reads
Other transactions see intermediate state.

### 2. Compensation Complexity
Some actions are hard to undo (e.g., sent email).

### 3. Ordering
Need to ensure compensations run in reverse order.

### Solution Patterns

| Problem | Solution |
|---------|----------|
| Dirty reads | Semantic locks, read-your-writes |
| Hard to undo | Plan for idempotent compensations |
| Tracking | Use saga state machine, event sourcing |

---

## 🛠️ Implementation Tools

| Tool | Type | Used By |
|------|------|---------|
| **Temporal** | Orchestration | Netflix, Uber, Stripe |
| **Camunda** | Orchestration | Enterprise |
| **Kafka + Debezium** | Choreography | Event-driven apps |
| **AWS Step Functions** | Orchestration | AWS-native |

---

## ✅ Key Takeaways

1. **Saga** = Sequence of local transactions with compensating actions
2. **Choreography**: Event-driven, decoupled but complex
3. **Orchestration**: Centralized coordinator, easier to reason about
4. **Compensations** must be idempotent and well-defined
5. **No isolation**: Intermediate state is visible
6. **Use for**: Long-running processes, microservices, high availability

---

[← Previous: 3PC](./03-three-phase-commit.md) | [Next: Concurrency Control →](./05-concurrency-control.md)
