# ZooKeeper

> The distributed coordination service that powers many systems.

---

## 🎯 What is ZooKeeper?

```mermaid
graph TB
    subgraph "ZooKeeper Use Cases"
        LE[Leader Election]
        Config[Config Management]
        Lock[Distributed Locks]
        SD[Service Discovery]
        Sync[Synchronization]
    end
```

**ZooKeeper provides primitives** for building distributed coordination.

---

## 🏗️ Architecture

```mermaid
graph TB
    subgraph "ZooKeeper Ensemble"
        L[Leader]
        F1[Follower]
        F2[Follower]
        
        L <-->|ZAB Protocol| F1
        L <-->|ZAB Protocol| F2
    end
    
    C1[Client] --> L
    C2[Client] --> F1
    C3[Client] --> F2
```

- **Leader**: Handles all writes
- **Followers**: Replicate data, serve reads
- **ZAB**: ZooKeeper Atomic Broadcast (like Paxos)

---

## 📁 Data Model: ZNodes

```mermaid
graph TB
    Root["/"]
    
    Root --> App["/app"]
    Root --> Config["/config"]
    
    App --> Leader["/app/leader"]
    App --> Workers["/app/workers"]
    
    Workers --> W1["/app/workers/w1"]
    Workers --> W2["/app/workers/w2"]
```

### ZNode Types

| Type | Description |
|------|-------------|
| **Persistent** | Survives client disconnect |
| **Ephemeral** | Deleted when client disconnects |
| **Sequential** | Auto-incrementing suffix |

---

## 🔧 Common Recipes

### Leader Election

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant ZK as ZooKeeper
    
    N1->>ZK: Create /election/node (ephemeral, sequential)
    ZK->>N1: Created /election/node-0001
    
    N2->>ZK: Create /election/node (ephemeral, sequential)
    ZK->>N2: Created /election/node-0002
    
    N1->>ZK: Get children of /election
    ZK->>N1: [node-0001, node-0002]
    
    Note over N1: I'm smallest = LEADER
    
    N2->>ZK: Watch node-0001
    Note over N2: If N1 dies, I become leader
```

### Distributed Lock

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant ZK as ZooKeeper
    
    C1->>ZK: Create /locks/resource (ephemeral)
    ZK->>C1: Success → LOCK ACQUIRED
    
    Note over C1: Do critical section...
    
    C1->>ZK: Delete /locks/resource
    Note over C1: LOCK RELEASED
```

---

## 📊 Guarantees

| Guarantee | Description |
|-----------|-------------|
| **Sequential consistency** | Updates applied in order |
| **Atomicity** | Updates succeed or fail entirely |
| **Durability** | Committed = persisted |
| **Timeliness** | Bounded staleness |

---

## 🔥 Real-World: Kafka + ZooKeeper

```mermaid
graph TB
    subgraph "Kafka Cluster"
        B1[Broker 1]
        B2[Broker 2]
        B3[Broker 3]
    end
    
    subgraph "ZooKeeper"
        ZK[ZK Ensemble]
    end
    
    B1 <--> ZK
    B2 <--> ZK
    B3 <--> ZK
    
    Note[ZK manages:<br/>- Broker metadata<br/>- Topic configs<br/>- Controller election]
```

**Note**: Kafka is moving away from ZK (KRaft mode).

---

## ⚠️ Limitations

| Issue | Impact |
|-------|--------|
| Not for large data | Max 1MB per znode |
| Latency for writes | All go through leader |
| Complex operations | Need application-level coordination |
| Separate cluster | Additional operational burden |

---

## ✅ Key Takeaways

1. **ZooKeeper** = Coordination primitives for distributed systems
2. **ZNodes** = Hierarchical namespace with ephemeral/sequential options
3. **Common uses**: Leader election, distributed locks, config management
4. **ZAB protocol** for consensus (like Paxos)
5. **Used by**: Kafka (legacy), HBase, Hadoop, Solr

---

[← Previous: Cassandra](./02-cassandra.md) | [Next: Spanner →](./04-spanner.md)
