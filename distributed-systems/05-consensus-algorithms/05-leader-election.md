# Leader Election

> Choosing a single coordinator in a distributed system.

---

## 🎯 The Problem

```mermaid
graph TB
    subgraph "Need for Leader"
        N1[Node 1]
        N2[Node 2]
        N3[Node 3]
        
        Q{Who coordinates?}
    end
    
    Q --> Leader[One leader needed!]
```

**Use cases**:
- Database primary selection
- Distributed lock holder
- Partition leader in Kafka
- Master in HDFS

---

## 📋 Requirements

| Property | Description |
|----------|-------------|
| **Safety** | At most one leader at any time |
| **Liveness** | Eventually a leader is elected |
| **Stability** | Leader stays unless it fails |

---

## 🔧 Bully Algorithm

The node with **highest ID** becomes leader.

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3 (highest)
    
    Note over N1: Detects leader failure
    N1->>N2: Election!
    N1->>N3: Election!
    
    N2->>N1: OK (I'm higher)
    N3->>N1: OK (I'm higher)
    
    N2->>N3: Election!
    N3->>N2: OK
    
    Note over N3: No higher node responds
    N3->>N1: I'm the leader!
    N3->>N2: I'm the leader!
```

### Algorithm Steps

1. Node detects leader failure
2. Sends ELECTION to all higher-numbered nodes
3. If no response → declare self leader
4. If OK received → wait for leader announcement

### Pros & Cons

| Pros | Cons |
|------|------|
| ✅ Simple | ❌ Many messages |
| ✅ Deterministic | ❌ Network partitions cause issues |

---

## 🔧 Ring Algorithm

Nodes organized in a logical ring.

```mermaid
graph TB
    N1[Node 1] --> N2[Node 2]
    N2 --> N3[Node 3]
    N3 --> N4[Node 4]
    N4 --> N1
    
    Note[Election message<br/>travels around ring]
```

### Algorithm

1. Node sends election message with its ID
2. Each node adds its ID and forwards
3. When originator receives it back → highest ID wins
4. Coordinator message sent around ring

---

## 🔧 Raft Leader Election

```mermaid
stateDiagram-v2
    Follower --> Candidate: Timeout,<br/>no heartbeat
    Candidate --> Leader: Wins election
    Candidate --> Follower: Loses election
    Leader --> Follower: Higher term seen
```

### Key Mechanism: Random Timeouts

```mermaid
graph TB
    subgraph "Election Timeouts"
        N1[Node 1: 150ms]
        N2[Node 2: 280ms]
        N3[Node 3: 200ms]
    end
    
    N1 -->|Timeout first!| Candidate[Becomes candidate]
    Candidate --> Election[Sends RequestVote]
```

**Random timeouts prevent split votes!**

---

## 🔧 ZooKeeper Leader Election

Using a coordination service:

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant ZK as ZooKeeper
    
    N1->>ZK: Create /election/node-0001 (ephemeral)
    N2->>ZK: Create /election/node-0002 (ephemeral)
    
    N1->>ZK: Get children of /election
    ZK->>N1: [node-0001, node-0002]
    
    Note over N1: I have lowest number!<br/>I'm the leader
    
    N2->>ZK: Watch node-0001
    Note over N2: If leader fails,<br/>I'll be notified
```

### ZooKeeper Recipe

1. Create **ephemeral sequential** znode under `/election`
2. Get all children
3. If your znode is smallest → you're leader
4. Otherwise, watch the next-smallest znode

---

## 🔥 Real-World: Redis Sentinel

```mermaid
graph TB
    subgraph "Redis Sentinel"
        S1[Sentinel 1]
        S2[Sentinel 2]
        S3[Sentinel 3]
        
        M[Redis Master]
        R1[Redis Replica]
        R2[Redis Replica]
    end
    
    S1 --> M
    S2 --> M
    S3 --> M
    
    M --> R1
    M --> R2
```

**How it works**:
1. Sentinels monitor master
2. If master fails, they vote on failover
3. Majority decides new master
4. Clients get new master from Sentinel

---

## ⚠️ Split-Brain Problem

```mermaid
graph TB
    subgraph "Partition A"
        L1[Old Leader]
        N1[Node 1]
    end
    
    subgraph "Partition B"
        L2[New Leader elected!]
        N2[Node 2]
        N3[Node 3]
    end
    
    Conflict[Two leaders!<br/>DATA CONFLICT]
    
    style Conflict fill:#f44336,color:#fff
```

### Solutions

| Solution | How |
|----------|-----|
| **Fencing tokens** | Stamps from leader to identify stale leaders |
| **Quorum** | Need majority to become leader |
| **Lease** | Leadership expires, must renew |

---

## 📊 Comparison

| Algorithm | Messages | Complexity | Split-brain safe? |
|-----------|----------|------------|-------------------|
| Bully | O(n²) | Low | ❌ No |
| Ring | O(n) | Low | ❌ No |
| Raft | O(n) | Medium | ✅ With quorum |
| ZooKeeper | O(n) | Low (uses ZK) | ✅ Yes |

---

## ✅ Key Takeaways

1. **Leader election** = choosing one coordinator
2. **Bully algorithm**: Highest ID wins (simple but chatty)
3. **Raft**: Random timeouts, majority vote
4. **ZooKeeper**: Ephemeral sequential znodes
5. **Split-brain** is the main danger — use quorums!
6. **Fencing tokens** prevent stale leaders from causing harm

---

[← Previous: Raft](./04-raft.md) | [Back to Module →](./README.md)
