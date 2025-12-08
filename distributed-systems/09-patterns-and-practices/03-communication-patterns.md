# Communication Patterns

> How services talk to each other in distributed systems.

---

## 📞 **Phone vs Email Analogy**

| Pattern | Like | Characteristics |
|---------|------|-----------------|
| **Synchronous** | Phone call | Wait for response |
| **Asynchronous** | Email | Send and continue |

---

## 🎯 Synchronous Communication

```mermaid
sequenceDiagram
    participant A as Service A
    participant B as Service B
    
    A->>B: Request
    Note over A: Waiting...
    B->>A: Response
    Note over A: Continue
```

### Request-Response (REST, gRPC)

```mermaid
graph LR
    Client -->|GET /users/1| Server
    Server -->|200 OK + data| Client
```

**Pros**: 
- Simple mental model
- Immediate feedback

**Cons**:
- Tight coupling
- Cascading failures
- Blocks the caller

---

## 📬 Asynchronous Communication

```mermaid
sequenceDiagram
    participant A as Service A
    participant Q as Message Queue
    participant B as Service B
    
    A->>Q: Publish message
    Note over A: Continue immediately
    Q->>B: Deliver message
    B->>B: Process
```

### Message Queues

```mermaid
graph LR
    Producer -->|Message| Queue[(Queue)]
    Queue -->|Message| Consumer
```

**Examples**: RabbitMQ, Amazon SQS, Redis

### Event Streaming

```mermaid
graph LR
    P1[Producer 1] --> Topic[Kafka Topic]
    P2[Producer 2] --> Topic
    Topic --> C1[Consumer Group 1]
    Topic --> C2[Consumer Group 2]
```

**Examples**: Kafka, Pulsar, Kinesis

---

## 📊 Sync vs Async Comparison

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| **Coupling** | Tight | Loose ✅ |
| **Latency** | Depends on downstream | Immediate return ✅ |
| **Failure handling** | Caller fails if callee fails | Retries possible ✅ |
| **Complexity** | Lower ✅ | Higher |
| **Debugging** | Easier ✅ | Harder |

---

## 🔄 Common Patterns

### 1. Request-Reply

```mermaid
graph LR
    A[Service A] -->|Request| B[Service B]
    B -->|Reply| A
```

### 2. Publish-Subscribe

```mermaid
graph TB
    Pub[Publisher] -->|Event| Topic[Topic/Channel]
    Topic --> Sub1[Subscriber 1]
    Topic --> Sub2[Subscriber 2]
    Topic --> Sub3[Subscriber 3]
```

**One-to-many**: Publisher doesn't know subscribers.

### 3. Point-to-Point Queue

```mermaid
graph LR
    P1[Producer] --> Q[(Queue)]
    P2[Producer] --> Q
    Q --> C[Consumer]
```

**Each message consumed once** by one consumer.

### 4. Event-Driven

```mermaid
graph TB
    Order[Order Service] -->|OrderPlaced| Bus[Event Bus]
    Bus --> Inventory[Inventory: Reserve stock]
    Bus --> Payment[Payment: Charge card]
    Bus --> Notification[Notification: Send email]
```

---

## 🍕 **Pizza Delivery Analogy**

| Pattern | Pizza Analogy |
|---------|---------------|
| **Sync** | Wait at counter for pizza |
| **Async (queue)** | Get buzzer, wait elsewhere |
| **Pub-sub** | Pizza tracker notifies all watchers |
| **Event-driven** | Each station starts when previous done |

---

## 🔧 Choosing the Right Pattern

```mermaid
graph TB
    Q1{Need immediate<br/>response?}
    
    Q1 -->|Yes| Sync[Synchronous<br/>REST, gRPC]
    Q1 -->|No| Q2{Multiple<br/>consumers?}
    
    Q2 -->|Yes| PubSub[Pub-Sub<br/>Kafka, SNS]
    Q2 -->|No| Queue[Queue<br/>SQS, RabbitMQ]
```

---

## 🔥 Real-World: Uber Architecture

```mermaid
graph TB
    subgraph "Uber Communications"
        REST[REST APIs<br/>User-facing]
        gRPC[gRPC<br/>Service-to-service]
        Kafka[Kafka<br/>Event streaming]
    end
    
    Mobile --> REST
    REST --> gRPC
    gRPC --> Kafka
```

**Uber uses all patterns**:
- REST for mobile apps
- gRPC between services
- Kafka for event-driven workflows

---

## ⚠️ Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| **Sync for everything** | Cascading failures | Add async where possible |
| **Fire and forget** | Lost messages | Use acknowledgments |
| **Huge messages** | Queue bloat | Use references, not data |
| **No idempotency** | Duplicate processing | Idempotent handlers |

---

## ✅ Key Takeaways

1. **Synchronous** = Wait for response (simpler, tighter coupling)
2. **Asynchronous** = Don't wait (resilient, complex)
3. **Pub-Sub** = One-to-many broadcast
4. **Queues** = One-to-one, processed once
5. **Mix patterns** based on requirements
6. **Always plan for failures** — retries, dead-letter queues

| Remember | Analogy |
|----------|---------|
| Sync | Phone call |
| Async | Email |
| Queue | DMV ticket |
| Pub-Sub | Newsletter |

---

[← Previous: Data Synchronization](./02-data-synchronization.md) | [Back to Module →](./README.md)
