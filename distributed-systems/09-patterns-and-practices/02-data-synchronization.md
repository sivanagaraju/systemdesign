# Data Synchronization Patterns

> Keeping data consistent across distributed services.

---

## 🎯 The Challenge

```mermaid
graph TB
    subgraph "Microservices"
        O[Order Service<br/>Order DB]
        I[Inventory Service<br/>Inventory DB]
        S[Shipping Service<br/>Shipping DB]
    end
    
    Q{How to keep<br/>data in sync?}
```

---

## 📤 Outbox Pattern

Ensure atomic writes to DB and message queue.

```mermaid
graph TB
    subgraph "Outbox Pattern"
        Service[Order Service]
        DB[(Order DB)]
        Outbox[(Outbox Table)]
        Poller[Outbox Poller]
        Queue[Message Queue]
        
        Service -->|1. Transaction| DB
        Service -->|1. Transaction| Outbox
        Poller -->|2. Poll| Outbox
        Poller -->|3. Publish| Queue
        Poller -->|4. Mark sent| Outbox
    end
```

**Key insight**: Write to outbox table in same DB transaction as business data!

```sql
BEGIN TRANSACTION;
  INSERT INTO orders (id, ...) VALUES (...);
  INSERT INTO outbox (event_type, payload) 
    VALUES ('OrderCreated', '{"id": 123}');
COMMIT;
```

---

## 🔄 Change Data Capture (CDC)

Capture database changes and stream them.

```mermaid
graph LR
    DB[(Database)]
    WAL[Transaction Log]
    CDC[CDC Tool<br/>Debezium]
    Kafka[Kafka]
    
    DB --> WAL
    WAL --> CDC
    CDC --> Kafka
    Kafka --> Consumer1[Service A]
    Kafka --> Consumer2[Service B]
```

**How it works**:
1. Database writes to transaction log (WAL)
2. CDC tool (e.g., Debezium) reads log
3. Converts to events, publishes to Kafka
4. Other services consume changes

---

## 📝 Event Sourcing

Store events, not current state.

```mermaid
graph TB
    subgraph "Traditional"
        T1[Order record<br/>status: shipped]
    end
    
    subgraph "Event Sourcing"
        E1[OrderCreated]
        E2[OrderPaid]
        E3[OrderShipped]
        
        E1 --> E2 --> E3
        E3 --> State[Derived state:<br/>status: shipped]
    end
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **Full audit log** | Every change recorded |
| **Time travel** | Replay to any point |
| **Event replay** | Rebuild views from events |
| **Decoupling** | Services react to events |

### Example: Bank Account

```python
# Events
events = [
    {"type": "AccountOpened", "amount": 0},
    {"type": "MoneyDeposited", "amount": 100},
    {"type": "MoneyWithdrawn", "amount": 30},
    {"type": "MoneyDeposited", "amount": 50}
]

# Derive current state
balance = 0
for event in events:
    if event["type"] == "MoneyDeposited":
        balance += event["amount"]
    elif event["type"] == "MoneyWithdrawn":
        balance -= event["amount"]
# balance = 120
```

---

## 🔀 CQRS (Command Query Responsibility Segregation)

Separate read and write models.

```mermaid
graph TB
    subgraph "CQRS"
        Cmd[Commands<br/>Write Model]
        Query[Queries<br/>Read Model]
        
        Write[(Write Store)]
        Read[(Read Store)]
        
        Cmd --> Write
        Write -->|Sync| Read
        Query --> Read
    end
```

**Why CQRS?**
- Different data shapes for reads vs writes
- Scale read and write independently
- Optimize each for its purpose

---

## 🔥 Real-World: LinkedIn Event Sourcing

```mermaid
graph TB
    subgraph "LinkedIn Architecture"
        Events[Activity Events]
        Kafka[(Kafka)]
        Search[Search Index]
        Feed[News Feed]
        Analytics[Analytics]
    end
    
    Events --> Kafka
    Kafka --> Search
    Kafka --> Feed
    Kafka --> Analytics
```

LinkedIn uses event sourcing for activity feeds!

---

## 📊 Pattern Comparison

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| **Outbox** | Reliable event publishing | Low |
| **CDC** | Stream all DB changes | Medium |
| **Event Sourcing** | Full audit, replay capability | High |
| **CQRS** | Different read/write needs | Medium |

---

## ✅ Key Takeaways

1. **Outbox pattern** for reliable DB + message atomicity
2. **CDC** streams database changes to other systems
3. **Event sourcing** stores events as source of truth
4. **CQRS** separates read and write models
5. **Often combined**: CDC + Outbox, Event Sourcing + CQRS

---

[← Previous: Failure Handling](./01-failure-handling.md) | [Next: Communication Patterns →](./03-communication-patterns.md)
