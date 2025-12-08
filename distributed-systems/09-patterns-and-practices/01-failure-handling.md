# Failure Handling Patterns

> Building resilient distributed systems that gracefully handle failures.

---

## 🎯 Why This Matters

In distributed systems, failures are **normal**, not exceptional.

```mermaid
graph TB
    subgraph "Things That Fail"
        N[Network]
        S[Services]
        D[Databases]
        H[Hardware]
    end
    
    Note[Plan for failure,<br/>not perfection!]
```

---

## 🔄 Retry Pattern

### Simple Retry

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Service
    
    C->>S: Request
    S--xC: ❌ Failure
    C->>C: Wait...
    C->>S: Retry
    S->>C: ✅ Success
```

### Exponential Backoff

```mermaid
graph LR
    R1[Retry 1<br/>Wait 1s] --> R2[Retry 2<br/>Wait 2s]
    R2 --> R3[Retry 3<br/>Wait 4s]
    R3 --> R4[Retry 4<br/>Wait 8s]
    R4 --> Max[Give up]
```

```python
def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError:
            wait = (2 ** attempt) + random.uniform(0, 1)  # Jitter
            time.sleep(wait)
    raise MaxRetriesExceeded()
```

**Jitter**: Random delay to prevent thundering herd.

---

## 🔌 Circuit Breaker

Prevent cascading failures by "breaking the circuit."

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: Failure threshold reached
    Open --> HalfOpen: Timeout expires
    HalfOpen --> Closed: Success
    HalfOpen --> Open: Failure
```

### States

| State | Behavior |
|-------|----------|
| **Closed** | Normal operation, count failures |
| **Open** | Fail fast, don't call service |
| **Half-Open** | Allow one test request |

```mermaid
sequenceDiagram
    participant C as Client
    participant CB as Circuit Breaker
    participant S as Service
    
    Note over CB: State: CLOSED
    C->>CB: Request
    CB->>S: Forward
    S--xCB: ❌ Failure
    
    Note over CB: Failures: 5/5<br/>State: OPEN
    
    C->>CB: Request
    CB->>C: ❌ FAIL FAST<br/>(no call to service)
    
    Note over CB: After timeout...<br/>State: HALF-OPEN
    
    C->>CB: Request
    CB->>S: Test request
    S->>CB: ✅ Success
    
    Note over CB: State: CLOSED
```

---

## ⏱️ Timeout Pattern

Never wait forever!

```mermaid
graph TB
    subgraph "Timeout Types"
        C[Connection Timeout<br/>Time to establish connection]
        R[Read Timeout<br/>Time to receive response]
        T[Total Timeout<br/>End-to-end limit]
    end
```

```python
# Good: Multiple timeout levels
requests.get(url, 
    connect_timeout=5,    # 5 seconds to connect
    read_timeout=30,      # 30 seconds to read
    total_timeout=60      # 60 seconds total
)
```

---

## 🌊 Bulkhead Pattern

Isolate failures to prevent system-wide impact.

```mermaid
graph TB
    subgraph "Without Bulkhead"
        S1[Slow Service] --> Pool[Shared Thread Pool]
        S2[Fast Service] --> Pool
        S3[Fast Service] --> Pool
        
        Pool -->|All threads stuck!| Blocked[System blocked]
    end
```

```mermaid
graph TB
    subgraph "With Bulkhead"
        S1[Slow Service] --> P1[Pool 1<br/>10 threads]
        S2[Fast Service] --> P2[Pool 2<br/>50 threads]
        S3[Fast Service] --> P3[Pool 3<br/>50 threads]
        
        Note[Slow service can't<br/>starve others!]
    end
```

---

## 🔙 Fallback Pattern

Provide degraded functionality instead of failure.

```mermaid
graph TB
    Request --> Primary[Primary Service]
    Primary -->|Success| Response
    Primary -->|Failure| Fallback[Fallback]
    
    Fallback --> Cache[Return cached data]
    Fallback --> Default[Return default value]
    Fallback --> Secondary[Call backup service]
```

---

## 🔥 Real-World: Netflix Hystrix

```mermaid
graph TB
    subgraph "Netflix Resilience"
        H[Hystrix<br/>(now Resilience4j)]
        
        H --> CB[Circuit Breaker]
        H --> BH[Bulkhead]
        H --> TO[Timeout]
        H --> FB[Fallback]
    end
```

**Netflix uses all these patterns** to handle 2B+ API requests/day!

---

## 📊 Pattern Summary

| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| **Retry** | Handle transient failures | Network blips, temporary issues |
| **Circuit Breaker** | Prevent cascading failures | Dependent service down |
| **Timeout** | Don't wait forever | Every remote call |
| **Bulkhead** | Isolate failures | Multiple dependencies |
| **Fallback** | Graceful degradation | When partial answers OK |

---

## ✅ Key Takeaways

1. **Retries with backoff + jitter** for transient failures
2. **Circuit breaker** to fail fast and prevent cascading failures
3. **Timeouts** on every external call
4. **Bulkheads** to isolate failure domains
5. **Fallbacks** for graceful degradation
6. **Combine patterns** for robust systems

---

[← Back to Module](./README.md) | [Next: Data Synchronization →](./02-data-synchronization.md)
