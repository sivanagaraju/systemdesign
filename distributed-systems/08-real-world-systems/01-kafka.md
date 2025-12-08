# Apache Kafka

> The distributed event streaming platform that handles trillions of events per day.

---

## 🎯 What is Kafka?

```mermaid
graph TB
    subgraph "Kafka Cluster"
        P1[Producer 1] --> T[Topic: orders]
        P2[Producer 2] --> T
        
        T --> Part1[Partition 0]
        T --> Part2[Partition 1]
        T --> Part3[Partition 2]
        
        Part1 --> C1[Consumer Group A]
        Part2 --> C1
        Part3 --> C1
        
        Part1 --> C2[Consumer Group B]
    end
```

**Kafka is a distributed commit log** — append-only, immutable, partitioned.

---

## 🏗️ Core Architecture

### Brokers

```mermaid
graph TB
    subgraph "Kafka Cluster"
        B1[Broker 1]
        B2[Broker 2]
        B3[Broker 3]
        
        ZK[ZooKeeper/KRaft]
        
        B1 <--> ZK
        B2 <--> ZK
        B3 <--> ZK
    end
```

### Topics & Partitions

```mermaid
graph TB
    subgraph "Topic: user-events"
        P0[Partition 0<br/>Messages 0-999]
        P1[Partition 1<br/>Messages 0-999]
        P2[Partition 2<br/>Messages 0-999]
    end
    
    subgraph "Partition 0 Detail"
        M0[Offset 0: Event A]
        M1[Offset 1: Event B]
        M2[Offset 2: Event C]
        M3[Offset 3: ...]
    end
```

**Key concepts**:
- **Topic**: Named feed of messages
- **Partition**: Ordered, immutable sequence
- **Offset**: Position in partition
- **Key**: Determines partition assignment

---

## 🔄 Replication

```mermaid
graph TB
    subgraph "Partition 0 Replicas"
        L[Leader<br/>Broker 1]
        F1[Follower<br/>Broker 2]
        F2[Follower<br/>Broker 3]
        
        L -->|Replicate| F1
        L -->|Replicate| F2
    end
    
    Producer -->|Write| L
    Consumer -->|Read| L
```

**ISR (In-Sync Replicas)**: Followers that are caught up with leader.

**Acknowledgment levels**:
| acks | Meaning | Durability |
|------|---------|------------|
| 0 | Fire and forget | None |
| 1 | Leader acknowledged | Medium |
| all | All ISRs acknowledged | Highest |

---

## 📊 Consumer Groups

```mermaid
graph TB
    subgraph "Topic with 3 Partitions"
        P0[P0]
        P1[P1]
        P2[P2]
    end
    
    subgraph "Consumer Group A (3 consumers)"
        C1[Consumer 1] --> P0
        C2[Consumer 2] --> P1
        C3[Consumer 3] --> P2
    end
    
    subgraph "Consumer Group B (2 consumers)"
        C4[Consumer 4] --> P0
        C4 --> P1
        C5[Consumer 5] --> P2
    end
```

**Rules**:
- Each partition consumed by ONE consumer in a group
- Consumer can consume multiple partitions
- Different groups get all messages independently

---

## 🔧 Distributed Systems Concepts in Kafka

### Partitioning (Sharding)

```mermaid
graph LR
    Msg[Message<br/>Key: user123] --> Hash[Hash Function]
    Hash --> Partition[Partition 2]
```

```java
// Partition = hash(key) % numPartitions
// Same key always goes to same partition (ordering!)
```

### Consistency

- **Within partition**: Total order guaranteed
- **Across partitions**: No ordering guarantee
- **Consumer offsets**: Stored in Kafka itself (__consumer_offsets topic)

### Fault Tolerance

```mermaid
sequenceDiagram
    participant P as Producer
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    P->>L: Write message
    L->>F1: Replicate
    L->>F2: Replicate
    F1->>L: ACK
    F2->>L: ACK
    L->>P: Success (if acks=all)
    
    Note over L: If leader fails...
    Note over F1: F1 becomes new leader
```

---

## 🔥 Real-World: LinkedIn Scale

```mermaid
graph TB
    subgraph "LinkedIn Kafka Usage"
        Events[7 trillion events/day]
        Data[4 PB data ingested/day]
        Clusters[100+ clusters]
    end
```

**Use cases at LinkedIn**:
- Activity tracking
- Metrics collection
- Log aggregation
- Stream processing

---

## 📋 When to Use Kafka

| Use Case | Why Kafka? |
|----------|-----------|
| Event streaming | High throughput, durability |
| Log aggregation | Append-only, retention policies |
| Stream processing | Kafka Streams, ksqlDB |
| Event sourcing | Immutable log as source of truth |
| Microservice communication | Decoupled, reliable |

---

## ⚠️ Kafka Trade-offs

| Pro | Con |
|-----|-----|
| ✅ High throughput | ❌ Complex to operate |
| ✅ Durable | ❌ Not for request-response |
| ✅ Horizontally scalable | ❌ No message priority |
| ✅ Replay capability | ❌ Ordering only per-partition |

---

## ✅ Key Takeaways

1. **Kafka = distributed commit log** with partitioning
2. **Partitions** enable parallelism and ordering within key
3. **Consumer groups** enable parallel processing
4. **Replication** with configurable durability (acks)
5. **Use for**: Event streaming, log aggregation, decoupling services
6. **Not for**: Request-response, low-latency messaging

---

[← Back to Module](./README.md) | [Next: Cassandra →](./02-cassandra.md)
