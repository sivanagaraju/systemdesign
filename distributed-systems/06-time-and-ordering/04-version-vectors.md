# Version Vectors

> Tracking versions per replica for conflict detection.

---

## 📦 **Package Tracking Analogy**

Imagine a package going through multiple sorting centers:
- Each center stamps the package with its ID and a counter
- `[NYC:3, LA:2, CHI:1]` = Processed 3 times in NYC, 2 in LA, 1 in Chicago

Version vectors work the same way for data!

---

## 🎯 What Problem Do They Solve?

```mermaid
graph TB
    subgraph "Multi-Master Replication"
        R1[Replica 1<br/>Updates: A, B]
        R2[Replica 2<br/>Updates: C, D]
        R3[Replica 3<br/>Updates: E]
    end
    
    Q{How to detect<br/>which updates<br/>happened when?}
```

---

## 📋 Version Vectors vs Vector Clocks

| Aspect | Vector Clocks | Version Vectors |
|--------|---------------|-----------------|
| **Tracks** | Events per process | Updates per replica |
| **Entries** | One per process | One per replica |
| **Size** | Can grow large | Bounded by replicas |
| **Use case** | Fine-grained causality | Replica synchronization |

---

## 🔧 How Version Vectors Work

### Initial State

```mermaid
graph TB
    subgraph "All Replicas"
        R1["Replica A<br/>VV: [A:0, B:0, C:0]<br/>Value: null"]
        R2["Replica B<br/>VV: [A:0, B:0, C:0]<br/>Value: null"]
        R3["Replica C<br/>VV: [A:0, B:0, C:0]<br/>Value: null"]
    end
```

### After Updates

```mermaid
sequenceDiagram
    participant A as Replica A
    participant B as Replica B
    
    Note over A: Write "Hello"<br/>VV: [A:1, B:0, C:0]
    A->>B: Sync [A:1, B:0, C:0]
    Note over B: VV: [A:1, B:0, C:0]<br/>Value: "Hello"
    
    Note over B: Write "World"<br/>VV: [A:1, B:1, C:0]
```

---

## 🔍 Comparing Version Vectors

```mermaid
graph TB
    subgraph "Comparison Rules"
        Equal["[A:2, B:1] = [A:2, B:1]<br/>EQUAL"]
        Dominates["[A:3, B:2] > [A:2, B:1]<br/>DOMINATES"]
        Concurrent["[A:2, B:1] || [A:1, B:2]<br/>CONCURRENT"]
    end
```

| V1 vs V2 | All V1 ≤ V2? | All V1 ≥ V2? | Result |
|----------|--------------|--------------|--------|
| Yes | Yes | EQUAL |
| Yes | No | V2 newer |
| No | Yes | V1 newer |
| No | No | CONCURRENT |

---

## 🔥 Real-World: Amazon DynamoDB

```mermaid
graph TB
    subgraph "DynamoDB Shopping Cart"
        C1["Cart from Replica 1<br/>VV: [R1:3, R2:1]<br/>Items: [iPhone]"]
        C2["Cart from Replica 2<br/>VV: [R1:2, R2:2]<br/>Items: [iPad]"]
        
        Compare{{"Compare VVs"}}
        
        C1 --> Compare
        C2 --> Compare
        
        Compare --> Concurrent["CONCURRENT!<br/>Return both to client"]
        Concurrent --> Merge["Client merges:<br/>[iPhone, iPad]"]
    end
```

---

## 🎭 **Family Photo Album Analogy**

| Scenario | Version Vector Equivalent |
|----------|---------------------------|
| Each family member adds photos | Each replica writes |
| Photos have "added by Mom, #5" | VV entry for that replica |
| Merging albums from grandparents | Comparing VVs to detect conflicts |
| Same photo from different sources | Concurrent updates |

---

## ✅ Key Takeaways

1. **Version vectors** track updates per replica (not per event)
2. **More compact** than vector clocks for replication
3. **Detect concurrent updates** for conflict resolution
4. **Used by**: DynamoDB, Riak, Cassandra (conceptually)
5. **Three outcomes**: One dominates, equal, or concurrent

---

[← Previous: Vector Clocks](./03-vector-clocks.md) | [Next: Causality →](./05-causality.md)
