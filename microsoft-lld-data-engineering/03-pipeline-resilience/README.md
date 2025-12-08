# 03 - Pipeline Resilience & Orchestration

> **Core LLD Skill:** Designing fault-tolerant data pipelines

Microsoft often asks: *"Your pipeline fails halfway through. How do you design it to recover gracefully?"*

---

## 📚 Topics in This Section

| File | Topic | Key Concepts |
|------|-------|--------------|
| [01-idempotency-design.md](./01-idempotency-design.md) | Idempotent Pipelines | Deduplication, processed files tracking |
| [02-checkpointing-recovery.md](./02-checkpointing-recovery.md) | Checkpoint Strategies | Resume vs restart, structured streaming |
| [03-dead-letter-queues.md](./03-dead-letter-queues.md) | Error Handling | DLQ design, quarantine tables |
| [04-azure-data-factory-deep-dive.md](./04-azure-data-factory-deep-dive.md) | ADF Internals | Integration runtimes, retry policies |

---

## 🎯 Common Interview Questions

1. *"If the pipeline runs twice, does it duplicate data?"*
2. *"If a 100GB load fails at 90%, how do you resume?"*
3. *"Where do bad records go? How do you reprocess them?"*
4. *"How do you handle transient failures in ADF?"*

---

## 🔑 Key Principles

| Principle | Description |
|-----------|-------------|
| **Idempotency** | Same input → same output (no duplicates on retry) |
| **Recoverability** | Resume from failure point, not restart from scratch |
| **Observability** | Know exactly where it failed and why |
| **Graceful Degradation** | Bad records don't stop the entire pipeline |

---

## 📖 Start Here

Begin with [Idempotency Design](./01-idempotency-design.md) to understand the foundation of reliable pipelines.
