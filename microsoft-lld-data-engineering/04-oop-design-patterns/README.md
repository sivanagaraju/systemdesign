# 04 - OOP Design Patterns for Data Engineers

> **Type B LLD:** Object-Oriented Design for Data Platform Utilities

Microsoft expects Data Engineers to write clean, maintainable code. They may ask you to design **classes and interfaces** for data platform utilities.

---

## 📚 Topics in This Section

| File | Topic | Key Concepts |
|------|-------|--------------|
| [01-class-design-fundamentals.md](./01-class-design-fundamentals.md) | SOLID Principles | Interface design, polymorphism |
| [02-singleton-pattern.md](./02-singleton-pattern.md) | Singleton | Config loader, connection pool |
| [03-factory-pattern.md](./03-factory-pattern.md) | Factory | Data source readers, writers |
| [04-custom-exceptions.md](./04-custom-exceptions.md) | Error Handling | Exception hierarchy |
| [05-rate-limiter-design.md](./05-rate-limiter-design.md) | Rate Limiter | API ingestion throttling |

---

## 🎯 Common Interview Questions

1. *"Design a custom Logging Library for our data platform"*
2. *"Design a Rate Limiter for an API ingestion script"*
3. *"Design a Data Source Factory that supports multiple sources"*
4. *"How would you structure exception classes for a data pipeline?"*

---

## 🔑 Key Concepts

| Concept | Application |
|---------|-------------|
| **Interfaces/ABC** | Define contracts for readers, writers, validators |
| **Polymorphism** | Same interface, different implementations |
| **Singleton** | Config loader, connection pool, logger |
| **Factory** | Create readers/writers based on source type |
| **Strategy** | Pluggable algorithms (validation, transformation) |

---

## 📖 Start Here

Begin with [Class Design Fundamentals](./01-class-design-fundamentals.md) for SOLID principles.
