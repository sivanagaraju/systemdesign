# HDFS and GFS (Distributed File Systems)

> Storing massive files across thousands of machines.

---

## 📚 **Library Analogy**

Imagine a book so large it can't fit on one shelf:

| Problem | Library Solution | Distributed FS |
|---------|------------------|----------------|
| Book too big | Split into volumes | Split into chunks |
| Volume might be lost | Keep copies | Replication |
| Finding a volume | Card catalog | Namenode/Master |

---

## 🎯 The Challenge

```mermaid
graph TB
    subgraph "Traditional Storage"
        File[Large File<br/>100 TB] --> Disk[Single Disk<br/>Max 10 TB ❌]
    end
    
    subgraph "Distributed FS"
        File2[Large File<br/>100 TB] --> C1[Chunk 1]
        File2 --> C2[Chunk 2]
        File2 --> C3[Chunk ...]
        File2 --> CN[Chunk N]
    end
```

---

## 🏗️ GFS/HDFS Architecture

```mermaid
graph TB
    subgraph "Master/NameNode"
        Master[Metadata Server<br/>File → Chunks<br/>Chunk → Locations]
    end
    
    subgraph "ChunkServers/DataNodes"
        CS1[Node 1<br/>Chunks: A, B, C]
        CS2[Node 2<br/>Chunks: A, D, E]
        CS3[Node 3<br/>Chunks: B, C, F]
    end
    
    Client --> Master
    Client --> CS1
    Client --> CS2
    Client --> CS3
```

---

## 📋 Key Design Decisions

### Large Chunk Size (64MB - 128MB)

| Small Chunks | Large Chunks |
|--------------|--------------|
| More metadata | Less metadata ✅ |
| More seeks | Fewer seeks ✅ |
| | Wastes space for small files |

### Write-Once, Read-Many

```mermaid
graph LR
    Write[Write once] --> Read1[Read]
    Write --> Read2[Read]
    Write --> Read3[Read many times]
```

**Why?** Simplifies consistency — no concurrent writes to same chunk.

### Replication (Default: 3 copies)

```mermaid
graph TB
    Chunk[Chunk A] --> R1[Replica 1<br/>Rack 1]
    Chunk --> R2[Replica 2<br/>Rack 1]
    Chunk --> R3[Replica 3<br/>Rack 2]
    
    Note[Different racks for<br/>failure tolerance]
```

---

## 🔄 Write Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant M as Master/NameNode
    participant D1 as DataNode 1
    participant D2 as DataNode 2
    participant D3 as DataNode 3
    
    C->>M: I want to write file X
    M->>C: Write to D1, D2, D3
    
    C->>D1: Data (chunk)
    D1->>D2: Pipeline replication
    D2->>D3: Pipeline replication
    
    D3->>D2: ACK
    D2->>D1: ACK
    D1->>C: Success
```

**Pipeline replication**: Data flows through chain, not from client to each.

---

## 📖 Read Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant M as Master/NameNode
    participant D as DataNode
    
    C->>M: Where is file X?
    M->>C: Chunks at D1, D2, D3
    C->>D: Read chunk (nearest)
    D->>C: Data
```

**Optimization**: Read from nearest replica!

---

## 🔥 Real-World: Google Scale

```mermaid
graph TB
    subgraph "GFS at Google (2003 paper)"
        Clusters[1000+ machines per cluster]
        Storage[Petabytes of data]
        Throughput[Aggregate 40 GB/s reads]
    end
```

---

## 📊 HDFS vs GFS Comparison

| Aspect | GFS (Google) | HDFS (Hadoop) |
|--------|--------------|---------------|
| Origin | Google (2003) | Open-source clone |
| Chunk size | 64 MB | 128 MB (default) |
| NameNode HA | Yes | Added later |
| Use | Google internal | Big Data ecosystem |

---

## ⚠️ Limitations

| Issue | Description |
|-------|-------------|
| **Single master** | Bottleneck, SPOF (mitigated in v2) |
| **Small files** | Inefficient (lots of metadata) |
| **Random writes** | Not supported |
| **Low latency** | Not designed for (batch processing) |

---

## 🏢 Modern Evolution

```mermaid
graph LR
    GFS1[GFS v1<br/>2003] --> Colossus[Colossus<br/>GFS v2]
    HDFS1[HDFS 1.x<br/>Single NameNode] --> HDFS2[HDFS 2.x<br/>HA NameNode]
    
    Alternatives[Cloud Storage<br/>S3, GCS, Azure Blob]
```

---

## ✅ Key Takeaways

1. **Split large files** into chunks across machines
2. **Replicate chunks** (default 3x) for durability
3. **Master/NameNode** stores metadata only
4. **Pipeline replication** for efficient writes
5. **Optimized for** large sequential reads/writes
6. **Not for** small files, random access, low latency

| Remember | Analogy |
|----------|---------|
| Chunks | Book volumes |
| NameNode | Card catalog |
| Replication | Multiple library copies |
| Pipeline write | Bucket brigade |

---

[← Previous: Kubernetes](./06-kubernetes.md) | [Back to Module →](./README.md)
