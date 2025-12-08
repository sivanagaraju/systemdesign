# Two-Phase Commit (2PC)

> The classic protocol for distributed atomic transactions.

---

## 🎯 The Problem

```mermaid
graph TB
    subgraph "Transfer $100"
        DB1[Bank A<br/>Deduct $100]
        DB2[Bank B<br/>Add $100]
    end
    
    Q{What if DB1 succeeds<br/>but DB2 fails?}
    
    style Q fill:#fff9c4
```

**We need atomicity across multiple databases!**

---

## 📋 The Protocol

### Phase 1: Prepare (Voting)

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P1 as Participant 1
    participant P2 as Participant 2
    
    C->>P1: PREPARE
    C->>P2: PREPARE
    
    Note over P1: Write to log,<br/>acquire locks
    Note over P2: Write to log,<br/>acquire locks
    
    P1->>C: VOTE YES ✅
    P2->>C: VOTE YES ✅
    
    Note over C: All voted YES!
```

### Phase 2: Commit (Decision)

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P1 as Participant 1
    participant P2 as Participant 2
    
    Note over C: All voted YES
    C->>P1: COMMIT
    C->>P2: COMMIT
    
    P1->>P1: Apply changes
    P2->>P2: Apply changes
    
    P1->>C: ACK
    P2->>C: ACK
    
    Note over C: Transaction complete!
```

### Abort Scenario

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P1 as Participant 1
    participant P2 as Participant 2
    
    C->>P1: PREPARE
    C->>P2: PREPARE
    
    P1->>C: VOTE YES ✅
    P2->>C: VOTE NO ❌
    
    Note over C: Someone voted NO!
    C->>P1: ABORT
    C->>P2: ABORT
    
    P1->>P1: Rollback
    P2->>P2: Rollback
```

---

## 🔒 Participant State Machine

```mermaid
stateDiagram-v2
    [*] --> Initial
    Initial --> Prepared: PREPARE received,<br/>vote YES
    Initial --> Aborted: PREPARE received,<br/>vote NO
    Prepared --> Committed: COMMIT received
    Prepared --> Aborted: ABORT received
    Committed --> [*]
    Aborted --> [*]
```

---

## ⚠️ The Blocking Problem

What if the **coordinator crashes** after PREPARE?

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P1 as Participant 1
    participant P2 as Participant 2
    
    C->>P1: PREPARE
    C->>P2: PREPARE
    
    P1->>C: VOTE YES
    P2->>C: VOTE YES
    
    Note over C: ❌ CRASH
    
    Note over P1: Holding locks...<br/>Can't commit or abort!<br/>BLOCKED!
    Note over P2: Holding locks...<br/>BLOCKED!
```

**Participants are stuck!** They can't decide without coordinator.

---

## 🔥 Real-World: XA Transactions

```mermaid
graph TB
    subgraph "XA (eXtended Architecture)"
        TM[Transaction Manager]
        RM1[Resource Manager 1<br/>MySQL]
        RM2[Resource Manager 2<br/>PostgreSQL]
        RM3[Resource Manager 3<br/>Message Queue]
    end
    
    TM -->|2PC| RM1
    TM -->|2PC| RM2
    TM -->|2PC| RM3
```

**XA standard** implements 2PC for distributed transactions across:
- Multiple databases
- Databases + message queues
- Different database vendors

---

## 📊 2PC Trade-offs

| Pros | Cons |
|------|------|
| ✅ Atomicity guaranteed | ❌ Blocking on coordinator failure |
| ✅ Widely supported (XA) | ❌ High latency (multiple round trips) |
| ✅ Strong consistency | ❌ Holds locks during protocol |
| ✅ Simple to understand | ❌ Single point of failure |

---

## 🔧 Handling Failures

### Coordinator Recovery

```mermaid
graph TB
    subgraph "Recovery from Log"
        Log[Transaction Log]
        
        Log -->|Commit logged| Action1[Send COMMIT to all]
        Log -->|Prepare logged| Action2[Ask participants]
        Log -->|Nothing logged| Action3[Abort transaction]
    end
```

### Participant Recovery

1. Check local log
2. If PREPARED but no decision → ask coordinator
3. If COMMITTED/ABORTED → replay decision

---

## ✅ Key Takeaways

1. **2PC** = Prepare phase + Commit phase
2. **All or nothing**: Either all commit or all abort
3. **Blocking protocol**: Coordinator failure blocks participants
4. **XA standard** implements 2PC across databases
5. **Use when**: Strong consistency required, can tolerate blocking
6. **Avoid when**: High availability required, long-running transactions

---

[← Back to Module](./README.md) | [Next: Three-Phase Commit →](./03-three-phase-commit.md)
