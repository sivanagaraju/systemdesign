# Microsoft Low Level Design (LLD) - Data Engineering Interview Prep

> **Round 2 Interview Preparation** for Data Engineer roles at Microsoft

This comprehensive guide covers both **Type A (Data-Specific LLD)** and **Type B (Standard SDE LLD)** concepts you'll encounter in Microsoft Data Engineering interviews.

---

## 📚 How to Use This Guide

1. **Start with fundamentals** - Components 1-5 cover core technical concepts
2. **Practice with scenarios** - Component 6 has real interview problems
3. **Master the framework** - Component 7 teaches you how to structure answers
4. **Deep dive as needed** - Components 8-13 cover advanced/specialized topics

---

## 🗂️ Content Index

### ⭐ START HERE: Architecture Design (Priority #1)

| # | Component | Key Topics |
|---|-----------|------------|
| 00 | [**Architecture Design**](./00-architecture-design/README.md) | **Lambda Architecture, How to Draw, Late-Arriving Data, PB-Scale** |

> This is the **most critical section** for Microsoft LLD interviews - covers end-to-end architecture and realistic questions.

---

### Core Technical Concepts (Type A - Data-Specific LLD)

| # | Component | Key Topics |
|---|-----------|------------|
| 01 | [Schema & Data Modeling](./01-schema-data-modeling/README.md) | Star/Snowflake, SCD, Partitioning, Cosmos DB |
| 02 | [Spark & Compute Internals](./02-spark-compute-internals/README.md) | Skew, Joins, Catalyst, File Formats, OOM |
| 03 | [Pipeline Resilience](./03-pipeline-resilience/README.md) | Idempotency, Checkpoints, DLQ, ADF |

### Software Engineering (Type B - Standard SDE LLD)

| # | Component | Key Topics |
|---|-----------|------------|
| 04 | [OOP Design Patterns](./04-oop-design-patterns/README.md) | SOLID, Singleton, Factory, Rate Limiter |

### Azure Platform Deep Dives

| # | Component | Key Topics |
|---|-----------|------------|
| 05 | [Azure Services LLD](./05-azure-services-lld/README.md) | ADLS Gen2, Delta Lake, Synapse |

### Interview Practice

| # | Component | Key Topics |
|---|-----------|------------|
| 06 | [Interview Scenarios](./06-interview-scenarios/README.md) | Real problems with complete solutions |
| 07 | [Answer Framework](./07-answer-framework/README.md) | How to structure your responses |

### Advanced Topics

| # | Component | Key Topics |
|---|-----------|------------|
| 08 | [Change Data Capture](./08-change-data-capture/README.md) | CDC patterns, MERGE, Incremental loads |
| 09 | [Data Lake Architecture](./09-data-lake-architecture/README.md) | Medallion, Schema evolution, Versioning |
| 10 | [ETL/ELT Patterns](./10-etl-elt-patterns/README.md) | Batch/Streaming, Data contracts, Events |
| 11 | [Testing & Monitoring](./11-testing-monitoring/README.md) | Testing, Observability, Alerting |
| 12 | [Security & Governance](./12-security-governance/README.md) | Encryption, Masking, Access control |
| 13 | [Advanced Data Modeling](./13-advanced-data-modeling/README.md) | Time-series, Graph, Multi-tenancy |

---

## 🎯 The Two Types of LLD Questions

### Type A: Data Component LLD (Most Common)
Focuses on internal mechanics of data pipelines, schemas, and transformations.

**Example:** *"Design the tables to store telemetry data for Xbox gaming sessions."*

**What they evaluate:**
- Schema design decisions (Star vs Snowflake)
- Partitioning strategies
- Performance optimization
- Fault tolerance

### Type B: Standard Software LLD (The Curveball)
Object-oriented design for data utilities.

**Example:** *"Design a custom Logging Library for our data platform."*

**What they evaluate:**
- Class structure and OOP principles
- Design patterns (Singleton, Factory)
- Error handling
- Clean code practices

---

## 🔑 Answer Framework (Quick Reference)

```
1. CLARIFY  → "Are we optimizing for reads or writes?"
2. DEFINE   → Draw schema or class structure
3. WALK     → Step through the logic flow
4. EDGE     → Handle failures and edge cases
```

See [Component 07](./07-answer-framework/README.md) for detailed guidance.

---

## 💡 Microsoft-Specific Topics to Master

| Service | Key LLD Concepts |
|---------|------------------|
| **Azure Databricks** | Spark internals, Delta Lake, Unity Catalog |
| **Azure Synapse** | Distribution strategies, PolyBase, Dedicated pools |
| **Azure Data Factory** | Integration Runtimes, Retry policies, Parameterization |
| **Cosmos DB** | Partition key selection, RU estimation |
| **ADLS Gen2** | Hierarchical namespace, ACLs, Performance tuning |

---

## 📖 Recommended Study Order

**Week 1:** Components 01, 02, 07 (Schema, Spark, Framework)
**Week 2:** Components 03, 04, 05 (Pipeline, OOP, Azure)
**Week 3:** Components 06, 08, 09 (Scenarios, CDC, Lake Arch)
**Week 4:** Components 10, 11, 12, 13 (Advanced topics)

---

*Good luck with your Microsoft interview! 🚀*
