# ADLS Gen2 Architecture

> **Interview Frequency:** ⭐⭐⭐⭐ (Azure-Specific)

## The Core Question

*"What's the difference between hierarchical and flat namespace in ADLS Gen2?"*

---

## 🏗️ Hierarchical vs Flat Namespace

### Flat Namespace (Blob Storage Style)

```
Container: data-lake
Objects: 
  raw/sales/2024/01/15/orders.parquet
  raw/sales/2024/01/15/returns.parquet
  raw/sales/2024/01/16/orders.parquet

→ These are just KEY-VALUE pairs, not real folders!
→ "raw/sales/2024/01/15/" is part of the blob NAME
```

### Hierarchical Namespace (HNS - True Directories)

```
Container: data-lake
└── raw/              ← Real directory
    └── sales/        ← Real directory
        └── 2024/
            └── 01/
                ├── 15/
                │   ├── orders.parquet
                │   └── returns.parquet
                └── 16/
                    └── orders.parquet
```

---

## ⚖️ Comparison

| Aspect | Flat Namespace | Hierarchical (HNS) |
|--------|----------------|-------------------|
| **Rename directory** | O(n) - copy all files | O(1) - metadata update |
| **Delete directory** | O(n) - delete each file | O(1) - single operation |
| **ACL inheritance** | Not supported | Children inherit ACLs |
| **Atomicity** | None | Atomic directory ops |
| **Performance** | Slower for directory ops | Faster |
| **Cost** | Slightly cheaper | Standard storage pricing |

---

## 🔐 Access Control

### RBAC vs ACL

```
┌─────────────────────────────────────────────────────────────┐
│                     ACCESS CONTROL                           │
├──────────────────────────┬──────────────────────────────────┤
│         RBAC             │             ACL                   │
│   (Role-Based Access)    │   (Access Control Lists)         │
├──────────────────────────┼──────────────────────────────────┤
│ Storage Account level    │ Container/Directory/File level   │
│ Coarse-grained          │ Fine-grained                      │
│ Storage Blob Data Reader│ rwx permissions per path          │
│ Storage Blob Data Owner │ User/group/other model            │
└──────────────────────────┴──────────────────────────────────┘
```

### ACL Example

```
Directory: /raw/pii-data/
├── ACL: 
│   owner: data-team       -> rwx
│   group: analytics-team  -> r-x
│   other:                -> ---
│
└── Files inherit parent ACL!
    └── customers.parquet  -> same permissions
```

---

## ⚡ Performance Best Practices

| Practice | Reason |
|----------|--------|
| **Enable HNS** | Atomic ops, faster directory ops |
| **Partition by date** | Prune reads to relevant data |
| **Target 256MB+ files** | Fewer API calls, better throughput |
| **Use Premium tier** | Low-latency analytics |
| **Co-locate compute** | Same region as storage |

---

## 📖 Next Topic

Continue to [Delta Lake Internals](./02-delta-lake-internals.md) for transaction log mechanics.
