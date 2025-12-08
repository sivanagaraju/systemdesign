# 📘 Module 2: Partitioning and Replication

> How distributed systems scale data and maintain availability.

---

## 📑 Contents

| File | Topic | Key Concepts |
|------|-------|--------------|
| [01-partitioning-strategies.md](./01-partitioning-strategies.md) | Data Partitioning | Vertical, Horizontal, Sharding |
| [02-partitioning-algorithms.md](./02-partitioning-algorithms.md) | Algorithms | Range, Hash, Consistent Hashing |
| [03-replication-fundamentals.md](./03-replication-fundamentals.md) | Replication Basics | Pessimistic vs Optimistic |
| [04-primary-backup-replication.md](./04-primary-backup-replication.md) | Leader-Follower | Sync/Async, Failover |
| [05-multi-primary-replication.md](./05-multi-primary-replication.md) | Multi-Master | Conflict Resolution |
| [06-quorums.md](./06-quorums.md) | Voting | Read/Write Quorums |

---

## 🎯 Learning Objectives

After completing this module, you will understand:
- ✅ How to split data across multiple nodes
- ✅ Trade-offs between partitioning algorithms
- ✅ How replication achieves availability
- ✅ Leader election and failover strategies
- ✅ Quorum-based consistency

---

## 🏢 Real Systems Using These Concepts

| Concept | Systems |
|---------|---------|
| Consistent Hashing | Cassandra, DynamoDB, Riak |
| Primary-Backup | PostgreSQL, MySQL, MongoDB |
| Multi-Primary | Cassandra, CockroachDB |
| Quorums | ZooKeeper, etcd, Consul |

---

## 🔗 Navigation
[← Previous: Fundamentals](../01-fundamentals/) | [Next: Consistency & CAP →](../03-consistency-and-cap/)
