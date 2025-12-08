# Raft Consensus Algorithm

> "Designed to be understandable" — The practical consensus algorithm.

---

## 🎯 Why Raft?

Paxos is notoriously hard to understand. Raft was designed with **understandability** as a primary goal.

```mermaid
graph LR
    Paxos[Paxos<br/>Complex, correct] --> Raft[Raft<br/>Understandable, correct]
    
    Note[Same guarantees,<br/>easier to implement]
```

---

## 📋 Raft Basics

### Node States

```mermaid
graph TB
    F[Follower] -->|Timeout, start election| C[Candidate]
    C -->|Wins election| L[Leader]
    C -->|Loses election| F
    L -->|Discovers higher term| F
    C -->|Discovers higher term| F
```

- **Follower**: Passive, responds to leader/candidates
- **Candidate**: Trying to become leader
- **Leader**: Handles all client requests, replicates log

---

## 🗳️ Leader Election

### Step by Step

```mermaid
sequenceDiagram
    participant F1 as Node 1 (Follower)
    participant F2 as Node 2 (Follower)
    participant F3 as Node 3 (Follower)
    
    Note over F1,F3: No heartbeat from leader...
    F1->>F1: Timeout! Become Candidate
    F1->>F1: Increment term to 2
    F1->>F1: Vote for self
    
    F1->>F2: RequestVote (term=2)
    F1->>F3: RequestVote (term=2)
    
    F2->>F1: Vote granted ✅
    F3->>F1: Vote granted ✅
    
    Note over F1: Majority! I'm the leader
    
    F1->>F2: Heartbeat (I'm leader of term 2)
    F1->>F3: Heartbeat (I'm leader of term 2)
```

### Election Rules

1. **Term**: Logical clock, increases with each election
2. **First come, first served**: Node votes for first valid request
3. **Majority wins**: Need > N/2 votes
4. **Random timeouts**: Prevents split votes

---

## 📝 Log Replication

### How Writes Work

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    C->>L: Write X=5
    L->>L: Append to log (uncommitted)
    
    L->>F1: AppendEntries(X=5)
    L->>F2: AppendEntries(X=5)
    
    F1->>L: Success
    F2->>L: Success
    
    Note over L: Majority confirmed!
    L->>L: Commit entry
    L->>C: Success ✅
    
    L->>F1: Commit notification
    L->>F2: Commit notification
```

### The Log

```mermaid
graph LR
    subgraph "Leader Log"
        LE1[Index 1<br/>Term 1<br/>X=1]
        LE2[Index 2<br/>Term 1<br/>Y=2]
        LE3[Index 3<br/>Term 2<br/>X=5]
    end
    
    subgraph "Follower Log"
        FE1[Index 1<br/>Term 1<br/>X=1]
        FE2[Index 2<br/>Term 1<br/>Y=2]
        FE3[Index 3<br/>Term 2<br/>X=5]
    end
    
    Note[Logs must match!]
```

---

## 🔒 Safety Guarantees

### 1. Election Safety

At most one leader per term.

```mermaid
graph TB
    T1[Term 1: Leader A]
    T2[Term 2: Leader B]
    T3[Term 3: Leader A]
    
    Note[Never two leaders<br/>in same term!]
```

### 2. Log Matching

If two logs have same index and term, all prior entries match.

### 3. Leader Completeness

If an entry is committed, it will be in all future leaders' logs.

---

## 🔄 Handling Failures

### Leader Crashes

```mermaid
sequenceDiagram
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    Note over L: ❌ CRASH
    
    F1->>F1: Timeout, no heartbeat
    F1->>F1: Become candidate
    F1->>F2: RequestVote
    F2->>F1: Vote granted
    
    Note over F1: F1 is new leader!
```

### Network Partition

```mermaid
graph TB
    subgraph "Majority Partition"
        L[New Leader]
        F1[Follower]
    end
    
    subgraph "Minority Partition"
        OldL[Old Leader<br/>Stepped down]
    end
    
    Note[Majority can elect<br/>new leader and proceed]
```

---

## 🏢 Real-World: etcd (Kubernetes)

```mermaid
graph TB
    subgraph "Kubernetes Control Plane"
        API[API Server]
        ETCD1[etcd 1]
        ETCD2[etcd 2]
        ETCD3[etcd 3]
        
        API --> ETCD1
        API --> ETCD2
        API --> ETCD3
        
        ETCD1 <-->|Raft| ETCD2
        ETCD2 <-->|Raft| ETCD3
        ETCD1 <-->|Raft| ETCD3
    end
    
    Note[All cluster config<br/>stored in etcd via Raft]
```

**Why Raft for Kubernetes?**
- Config must be consistent
- Tolerates node failures (2 of 3, 3 of 5, etc.)
- Understandable = maintainable

---

## 📊 Raft vs Paxos

| Aspect | Raft | Paxos |
|--------|------|-------|
| Understandability | ✅ Designed for it | ❌ Notoriously complex |
| Leader | Always has one | Optional |
| Log structure | Explicit | Implicit |
| Membership changes | Built-in | Separate |
| Production use | etcd, Consul | Chubby, Spanner |

---

## 🔧 Key Implementation Details

### Heartbeat & Timeout

```
Leader heartbeat: 50-100ms
Election timeout: 150-300ms (random)
```

Random timeout prevents multiple candidates simultaneously.

### Cluster Sizes

| Nodes | Tolerated Failures | Quorum |
|-------|-------------------|--------|
| 3 | 1 | 2 |
| 5 | 2 | 3 |
| 7 | 3 | 4 |

---

## ✅ Key Takeaways

1. **Raft** = understandable consensus (vs Paxos)
2. **Three states**: Follower → Candidate → Leader
3. **Leader handles all writes**, replicates log
4. **Majority quorum** required for election and commits
5. **Random timeouts** prevent split votes
6. **Used by**: etcd, Consul, CockroachDB

---

[← Previous: Paxos](./03-paxos.md) | [Next: Leader Election →](./05-leader-election.md)
