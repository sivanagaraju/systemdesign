# Kubernetes

> The container orchestration platform that powers modern distributed systems.

---

## 🚢 **Shipping Port Analogy**

| Kubernetes | Shipping Port |
|------------|---------------|
| Pods | Shipping containers |
| Nodes | Ships |
| Control Plane | Port authority |
| Scheduler | Crane operator deciding where to put containers |
| etcd | Master manifest of all containers |

---

## 🎯 What Kubernetes Does

```mermaid
graph TB
    subgraph "You Declare"
        Desired[Desired State<br/>3 replicas of my app]
    end
    
    subgraph "Kubernetes Does"
        K8s[Kubernetes]
        Monitor[Monitors actual state]
        Reconcile[Reconciles differences]
        K8s --> Monitor --> Reconcile
    end
    
    Desired --> K8s
    Reconcile -->|Maintains| Desired
```

---

## 🏗️ Architecture

```mermaid
graph TB
    subgraph "Control Plane"
        API[API Server]
        ETCD[(etcd)]
        Sched[Scheduler]
        CM[Controller Manager]
    end
    
    subgraph "Worker Nodes"
        N1[Node 1]
        N2[Node 2]
        N3[Node 3]
    end
    
    API <--> ETCD
    API --> Sched
    API --> CM
    API --> N1
    API --> N2
    API --> N3
```

### Key Components

| Component | Role |
|-----------|------|
| **API Server** | Frontend, all communication |
| **etcd** | Distributed key-value store (Raft!) |
| **Scheduler** | Assigns pods to nodes |
| **Controller Manager** | Runs reconciliation loops |
| **kubelet** | Agent on each node |

---

## 📦 Core Concepts

### Pod (Smallest Deployable Unit)

```mermaid
graph TB
    subgraph "Pod"
        C1[Container 1]
        C2[Container 2]
        Vol[(Shared Volume)]
        
        C1 --> Vol
        C2 --> Vol
    end
    
    Net[Shared Network<br/>Same IP]
    Pod --> Net
```

### Deployment (Manages Pods)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:1.0
```

### Service (Stable Networking)

```mermaid
graph TB
    Svc[Service<br/>my-app:80] --> P1[Pod 1]
    Svc --> P2[Pod 2]
    Svc --> P3[Pod 3]
    
    Client --> Svc
```

---

## 🔄 Self-Healing

```mermaid
sequenceDiagram
    participant K as Kubernetes
    participant N as Node
    participant P as Pod
    
    Note over P: ❌ Pod crashes
    K->>K: Detect pod failure
    K->>N: Schedule new pod
    N->>K: Pod running ✅
    
    Note over K: Desired state restored!
```

---

## 🌐 Distributed Systems Concepts in K8s

| Concept | Kubernetes Implementation |
|---------|---------------------------|
| **Consensus** | etcd (Raft) |
| **Service discovery** | DNS, Services |
| **Load balancing** | Services, Ingress |
| **Configuration** | ConfigMaps, Secrets |
| **Health checks** | Liveness, Readiness probes |

---

## 🔥 Real-World: Spotify

```mermaid
graph TB
    subgraph "Spotify on K8s"
        Clusters[150+ clusters]
        Services[500+ microservices]
        Deploys[250+ deploys/day]
    end
```

---

## ✅ Key Takeaways

1. **Declarative** = Tell K8s desired state, it maintains it
2. **etcd** = Raft-based distributed storage
3. **Self-healing** = Automatic recovery from failures
4. **Service discovery** built-in
5. **Perfect for** microservices, stateless apps

| Remember | Analogy |
|----------|---------|
| Pod | Shipping container |
| Deployment | Order for X containers |
| Service | Address book entry |
| etcd | Master manifest |

---

[← Previous: DynamoDB](./05-dynamodb.md) | [Next: HDFS/GFS →](./07-hdfs-gfs.md)
