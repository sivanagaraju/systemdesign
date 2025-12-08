# What is a Distributed System?

> "A distributed system is a system whose components are located on different networked computers, which communicate and coordinate their actions by passing messages to one another." — Coulouris et al.

---

## 🏗️ Architecture Overview

```mermaid
graph TB
    subgraph "Distributed System"
        N1[Node A<br/>Web Server]
        N2[Node B<br/>App Server]
        N3[Node C<br/>Database]
        N4[Node D<br/>Cache]
    end
    
    Client((Client)) --> N1
    N1 <-->|Messages| N2
    N2 <-->|Messages| N3
    N2 <-->|Messages| N4
    N3 <-->|Replication| N3R[DB Replica]
    
    style N1 fill:#e1f5fe
    style N2 fill:#fff3e0
    style N3 fill:#e8f5e9
    style N4 fill:#fce4ec
```

---

## 🔑 Key Components

| Component | Description | Example |
|-----------|-------------|---------|
| **Nodes** | Individual machines running software | Web servers, databases |
| **Network** | Communication medium between nodes | TCP/IP, HTTP, gRPC |
| **Messages** | Data exchanged between nodes | API calls, events |
| **Coordination** | Logic to keep nodes in sync | Leader election, consensus |

---

## 💪 Why Use Distributed Systems?

### 1. **Performance**
Single machines have physical limits. Distributed systems achieve higher throughput by parallelizing work.

```mermaid
graph LR
    subgraph "Single Machine"
        S1[100 req/s max]
    end
    
    subgraph "Distributed (3 nodes)"
        D1[100 req/s]
        D2[100 req/s]
        D3[100 req/s]
    end
    
    LB[Load Balancer<br/>300 req/s] --> D1
    LB --> D2
    LB --> D3
```

### 2. **Scalability**
- **Vertical Scaling**: Add more resources (CPU, RAM) to one machine — has limits
- **Horizontal Scaling**: Add more machines — virtually unlimited

### 3. **Availability**
Redundancy ensures the system survives failures.

> **Five-nines (99.999%)** = Only 5.26 minutes downtime/year

---

## 🌍 Real-World Example: Netflix

```mermaid
graph TB
    User((User)) --> CDN[CDN<br/>Edge Servers]
    CDN --> API[API Gateway]
    API --> Auth[Auth Service]
    API --> Catalog[Catalog Service]
    API --> Streaming[Streaming Service]
    
    Catalog --> CassDB[(Cassandra)]
    Auth --> MySQL[(MySQL)]
    Streaming --> S3[(S3 Storage)]
    
    style CDN fill:#ff9800
    style CassDB fill:#2196f3
    style S3 fill:#4caf50
```

**Why Netflix is distributed:**
- 200+ million subscribers globally
- Thousands of servers across regions
- No single point of failure
- Content served from nearest edge location

---

## ⚠️ The Trade-offs

| Benefit | Challenge |
|---------|-----------|
| Higher performance | Network latency |
| Better availability | Partial failures |
| Elastic scalability | Data consistency |
| Geographic distribution | Coordination overhead |

---

## 🔥 Real-World Incident: Single Point of Failure

### Facebook Outage (October 2021)
- **What happened**: DNS servers became unreachable due to a configuration change
- **Impact**: 6+ hours of global outage affecting 3.5 billion users
- **Lesson**: Even with distributed systems, centralized dependencies (like DNS) can be SPOFs

---

## ✅ Key Takeaways

1. **Distributed systems** = multiple computers working as one logical system
2. **Three main benefits**: Performance, Scalability, Availability
3. **Core mechanism**: Message passing between nodes
4. **Trade-off**: Complexity increases significantly
5. **Real systems**: Netflix, Google, Amazon all rely on distributed architecture

---

## 📚 Further Reading
- [Distributed Systems for Fun and Profit](http://book.mixu.net/distsys/)
- [Designing Data-Intensive Applications](https://dataintensive.net/)

---

[← Back to Module](./README.md) | [Next: Fallacies of Distributed Computing →](./02-fallacies-of-distributed-computing.md)
