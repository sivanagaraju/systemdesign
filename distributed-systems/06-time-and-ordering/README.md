# 📘 Module 6: Time and Ordering

> How distributed systems reason about time and ordering of events.

---

## 📑 Contents

| File | Topic | Key Concepts |
|------|-------|--------------|
| [01-physical-vs-logical-time.md](./01-physical-vs-logical-time.md) | Time Types | Physical clocks vs Logical clocks |
| [02-lamport-clocks.md](./02-lamport-clocks.md) | Lamport Clocks | Scalar logical time |
| [03-vector-clocks.md](./03-vector-clocks.md) | Vector Clocks | Detecting causality |
| [04-hybrid-logical-clocks.md](./04-hybrid-logical-clocks.md) | HLC | Best of both worlds |

---

## 🎯 Learning Objectives

After completing this module, you will understand:
- ✅ Why physical time is unreliable in distributed systems
- ✅ How Lamport clocks establish ordering
- ✅ How vector clocks detect causality
- ✅ Real-world applications of these concepts

---

## 🏢 Where These Are Used

| Concept | Systems |
|---------|---------|
| Lamport clocks | Paxos, Raft (term numbers) |
| Vector clocks | Riak, DynamoDB (conflict detection) |
| HLC | CockroachDB, MongoDB |
| TrueTime | Google Spanner |

---

## 🔗 Navigation
[← Previous: Consensus Algorithms](../05-consensus-algorithms/) | [Next: Networking & Security →](../07-networking-and-security/)
