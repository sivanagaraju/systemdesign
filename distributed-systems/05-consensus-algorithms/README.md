# 📘 Module 5: Consensus Algorithms

> How distributed systems agree on a single value — even with failures.

---

## 📑 Contents

| File | Topic | Key Concepts |
|------|-------|--------------|
| [01-consensus-problem.md](./01-consensus-problem.md) | The Problem | Why consensus is hard |
| [02-flp-impossibility.md](./02-flp-impossibility.md) | FLP | Impossibility result |
| [03-paxos.md](./03-paxos.md) | Paxos | Classic algorithm |
| [04-raft.md](./04-raft.md) | Raft | Understandable consensus |
| [05-leader-election.md](./05-leader-election.md) | Leader Election | Practical application |

---

## 🎯 Learning Objectives

After completing this module, you will understand:
- ✅ What the consensus problem is
- ✅ Why it's impossible to solve perfectly (FLP)
- ✅ How Paxos and Raft work
- ✅ How to implement leader election

---

## 🏢 Real Systems Using Consensus

| System | Algorithm | Use Case |
|--------|-----------|----------|
| ZooKeeper | ZAB (Paxos-like) | Distributed coordination |
| etcd | Raft | Kubernetes config |
| Consul | Raft | Service discovery |
| CockroachDB | Multi-Raft | Distributed SQL |
| Google Spanner | Paxos | Global transactions |

---

## 🔗 Navigation
[← Previous: Distributed Transactions](../04-distributed-transactions/) | [Next: Time & Ordering →](../06-time-and-ordering/)
