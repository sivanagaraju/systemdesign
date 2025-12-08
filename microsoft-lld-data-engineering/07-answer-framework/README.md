# 07 - Answer Framework

> **How to structure your LLD interview answers**

A structured answer shows clear thinking and communication skills.

---

## 📚 Topics in This Section

| File | Topic |
|------|-------|
| [01-clarification-questions.md](./01-clarification-questions.md) | Questions to ask before designing |
| [02-edge-cases-checklist.md](./02-edge-cases-checklist.md) | The "Microsoft touch" - what to cover |

---

## 🎯 The 4-Step Framework

```
┌─────────────────────────────────────────────────────────────┐
│                    LLD ANSWER FLOW                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. CLARIFY  ──►  "Are we optimizing for reads or writes?" │
│                   "What's the scale - rows per day?"        │
│                                                             │
│  2. DEFINE   ──►  Draw schema or class structure            │
│                   Write key attributes/columns              │
│                                                             │
│  3. WALK     ──►  "First, I read the source..."            │
│                   "Then, I validate..."                     │
│                   "Finally, I write to..."                  │
│                                                             │
│  4. EDGE     ──►  "What if the file is empty?"             │
│                   "What if schema changes?"                 │
│                   "What if we hit rate limits?"             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## ⏱️ Time Allocation (45-min LLD)

| Phase | Time | What To Do |
|-------|------|------------|
| **Clarify** | 5 min | Ask questions, confirm scope |
| **High-Level** | 10 min | Draw diagram, identify components |
| **Detailed Design** | 20 min | Schema, code, algorithms |
| **Edge Cases** | 10 min | Failures, scale, security |

---

## 📖 Start Here

Begin with [Clarification Questions](./01-clarification-questions.md) to know what to ask.
