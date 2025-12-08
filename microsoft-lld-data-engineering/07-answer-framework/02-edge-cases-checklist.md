# Edge Cases Checklist

> **The "Microsoft Touch"** - what differentiates good from great answers

Always address these edge cases to impress your interviewer.

---

## 📋 Universal Edge Cases

| Category | Questions to Address |
|----------|---------------------|
| **Empty data** | What if the file/table is empty? |
| **Null values** | How do we handle NULLs in join keys? |
| **Duplicates** | What if source sends duplicates? |
| **Schema drift** | What if a new column appears? |
| **Large values** | What if one record is 100MB? |
| **Concurrency** | What if two pipelines run simultaneously? |

---

## 🔄 Pipeline Edge Cases

| Scenario | Your Answer Should Include |
|----------|---------------------------|
| **Retry after failure** | Idempotency - same result on rerun |
| **Partial failure** | Checkpointing - resume from last point |
| **Bad records** | DLQ - quarantine and continue |
| **Late data** | Watermarking or reprocessing strategy |
| **Schema mismatch** | Validation before processing |

---

## ⚡ Performance Edge Cases

| Scenario | Your Answer Should Include |
|----------|---------------------------|
| **Data skew** | Salting technique or AQE |
| **OOM error** | Increase memory or reduce partitions |
| **Slow queries** | Check for broadcast join opportunity |
| **Small files** | OPTIMIZE command, auto-compact |

---

## 🔐 Security Edge Cases

| Scenario | Your Answer Should Include |
|----------|---------------------------|
| **PII data** | Masking, encryption at rest |
| **Multi-tenant** | Row-level security, isolation |
| **Audit** | Log who accessed what, when |
| **Key rotation** | Key vault, automated rotation |

---

## 🎯 How to Bring Up Edge Cases

> *"Now let me address some edge cases:*
> - *What if the file is empty? I'd log and skip, not fail.*
> - *What if schema changes? I've enabled schema evolution.*
> - *What if we hit rate limits? Exponential backoff with retries."*

This shows you think beyond the happy path!

---

## 🏆 The Microsoft Touch

These topics specifically impress Microsoft interviewers:

1. **Idempotency** - Always mention for pipelines
2. **ACID guarantees** - Show you understand Delta Lake
3. **Cost optimization** - Not just correctness, but efficiency
4. **Observability** - How do you know it's working?
5. **Security** - PII, encryption, access control

---

*Good luck with your interview! 🚀*
