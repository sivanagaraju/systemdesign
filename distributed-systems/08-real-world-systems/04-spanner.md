# Google Spanner

> The globally distributed database with strong consistency.

---

## 🎯 What Makes Spanner Special?

```mermaid
graph TB
    subgraph "Spanner's Unique Combo"
        Global[Global Distribution]
        Strong[Strong Consistency]
        SQL[SQL Transactions]
        
        Global --> Magic[Usually impossible!<br/>But Spanner does it]
        Strong --> Magic
        SQL --> Magic
    end
    
    style Magic fill:#4caf50,color:#fff
```

---

## 🏗️ Architecture

```mermaid
graph TB
    subgraph "Global Spanner"
        US[US Datacenter]
        EU[Europe Datacenter]
        Asia[Asia Datacenter]
        
        US <-->|Paxos| EU
        EU <-->|Paxos| Asia
        US <-->|Paxos| Asia
    end
    
    subgraph "Each DC"
        Leaders[Paxos Leaders]
        Replicas[Replicas]
    end
```

---

## ⏱️ TrueTime: The Secret Sauce

```mermaid
graph TB
    subgraph "TrueTime API"
        GPS[GPS Receivers]
        Atomic[Atomic Clocks]
        
        GPS --> TT[TrueTime Service]
        Atomic --> TT
        
        TT --> API["TrueTime.Now()<br/>Returns: [earliest, latest]"]
    end
```

### How TrueTime Works

```python
interval = TrueTime.now()
# interval.earliest = 10:00:00.000
# interval.latest   = 10:00:00.007
# Actual time is GUARANTEED within this range
```

**Bound uncertainty**: Usually < 7ms!

---

## 🔒 External Consistency

```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant S as Spanner
    participant T2 as Transaction 2
    
    T1->>S: Commit (timestamp: 100)
    Note over S: Wait for uncertainty to pass
    S->>T1: Committed
    
    Note over T1,T2: T1 returns to client
    Note over T1,T2: Client tells another app
    
    T2->>S: Start transaction
    Note over S: Timestamp: 110 (guaranteed > 100)
    T2->>S: Read
    S->>T2: Sees T1's write ✅
```

**If T1 commits before T2 starts (real time), T2 sees T1's effects.**

---

## 📊 Data Model

```mermaid
graph TB
    subgraph "Spanner Schema"
        DB[Database]
        Table1[Parent Table<br/>e.g., Customers]
        Table2[Child Table<br/>e.g., Orders]
        
        DB --> Table1
        Table1 -->|Interleaved| Table2
    end
```

**Interleaving**: Store related rows together for locality.

---

## 🔧 Transactions

```sql
-- Spanner supports standard SQL
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**Spanner provides**:
- Serializable isolation
- Global strong consistency
- Automatic sharding

---

## 📊 Spanner vs Other DBs

| Feature | Spanner | Cassandra | DynamoDB |
|---------|---------|-----------|----------|
| Consistency | Strong | Eventual | Eventual/Strong |
| SQL | ✅ Full | ❌ CQL | ❌ Limited |
| Global | ✅ Yes | ✅ Yes | ⚠️ Per region |
| Open source | ❌ No | ✅ Yes | ❌ No |

---

## 🔥 Real-World: Google Services

```mermaid
graph TB
    subgraph "Google Uses Spanner For"
        Ads[Google Ads<br/>Billions $/day]
        Play[Google Play<br/>App purchases]
        Gmail[Gmail<br/>Metadata]
    end
```

---

## ⚠️ Trade-offs

| Pro | Con |
|-----|-----|
| ✅ Strong consistency | ❌ Higher latency (wait for uncertainty) |
| ✅ Global transactions | ❌ Proprietary (Cloud Spanner) |
| ✅ SQL support | ❌ Expensive at scale |
| ✅ Automatic sharding | ❌ Requires GPS/atomic clocks |

---

## ✅ Key Takeaways

1. **Spanner** = Globally distributed + strongly consistent + SQL
2. **TrueTime** enables external consistency with bounded uncertainty
3. **Paxos** for replication across datacenters
4. **Wait out uncertainty** for timestamp ordering
5. **Use when**: Global transactions required, consistency critical

---

[← Previous: ZooKeeper](./03-zookeeper.md) | [Next: DynamoDB →](./05-dynamodb.md)
