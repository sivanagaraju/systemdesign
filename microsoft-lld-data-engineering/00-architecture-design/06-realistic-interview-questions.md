# Realistic Interview Questions

> **Actual questions Microsoft interviewers ask** with expected answers

Practice these out loud! The key is structured, confident delivery.

---

## 📋 Question Bank by Category

### Category 1: Architecture Design

---

#### Q1: "*Design an architecture for IoT sensor data that supports both real-time dashboards and historical analytics*"

**What they're testing:** Lambda Architecture, Medallion pattern, streaming + batch

**Expected answer structure:**
```
1. Draw sources (IoT devices) → Event Hub
2. Show two paths:
   - Speed Layer: Event Hub → Spark Streaming → Real-time Delta table → Dashboard
   - Batch Layer: Event Hub → Bronze → Silver → Gold → Analytics
3. Explain serving layer merges both
4. Mention: checkpointing, watermarks, partition by time
```

**Key phrases to use:**
- *"I'd use Lambda architecture with batch and speed layers"*
- *"Bronze-Silver-Gold for data quality progression"*
- *"Watermarking handles late-arriving events"*

---

#### Q2: "*How do you handle late-arriving dimension data?*"

**What they're testing:** Data integration patterns, staging, retry logic

**Expected answer:**
- *"I'd implement a Staging + Retry pattern"*
- *"LEFT JOIN to identify unmatched records"*
- *"Staging table for unmatched, hourly retry job"*
- *"After 24 hours, move to DLQ and alert"*
- *"Alternative: Unknown member pattern with backfill"*

---

#### Q3: "*You're getting petabytes of data daily. How do you design for this?*"

**What they're testing:** Scale considerations, optimization

**Expected answer:**
- *"Partition by time (year/month/day/hour)"*
- *"Incremental processing with watermarks"*
- *"Never full table scan - always filter on partition first"*
- *"File size targets: 256MB-1GB, avoid small files"*
- *"OPTIMIZE and Z-ORDER for query acceleration"*
- *"Autoscaling clusters with spot instances"*

---

### Category 2: Pipeline Design

---

#### Q4: "*How do you know the latest data has arrived?*"

**What they're testing:** Data freshness monitoring, observability

**Expected answer:**
- *"Watermark tracking - store MAX(event_time) processed"*
- *"Freshness checks comparing watermark to current time"*
- *"Alert if data is older than SLA (e.g., 30 minutes)"*
- *"Auto Loader checkpoints track last processed files"*

---

#### Q5: "*What happens if your batch job fails at 90%?*"

**What they're testing:** Idempotency, checkpointing, recovery

**Expected answer:**
- *"Idempotent design - can restart without duplicates"*
- *"Checkpoint tracks last successful offset"*
- *"Resume from checkpoint, not from beginning"*
- *"For Delta: atomic writes ensure no partial data"*
- Code mention: `df.write.format("delta").option("replaceWhere", "...")` 

---

#### Q6: "*How do you make a pipeline idempotent?*"

**What they're testing:** Critical DE concept

**Expected answer:**
- *"Same input always produces same output"*
- *"Track processed files/offsets in metadata table"*
- *"Partition overwrite mode - replace entire partition"*
- *"MERGE for upsert - handles duplicates"*
- *"Checksum/hash to detect already-processed files"*

---

### Category 3: Data Quality

---

#### Q7: "*How do you handle bad data without stopping the pipeline?*"

**What they're testing:** DLQ pattern, data quality

**Expected answer:**
- *"Validate at Bronze → Silver transition"*
- *"Split: valid records proceed, invalid go to quarantine"*
- *"Quarantine table stores raw record + error reason"*
- *"Retry job attempts reprocessing"*
- *"Alert on quarantine count threshold"*

---

#### Q8: "*When do you use Star Schema vs Snowflake?*"

**What they're testing:** Data modeling

**Expected answer:**
- *"Star: query performance, fewer joins, BI tools"*
- *"Snowflake: storage efficiency, dimension reuse"*
- *"I prefer Star for analytics workloads"*
- *"At Microsoft scale, storage is cheap, query speed matters"*

---

### Category 4: CDC & Updates

---

#### Q9: "*How do you implement CDC in your architecture?*"

**What they're testing:** Change data capture understanding

**Expected answer:**
```
Source DB (SQL Server)
       │
       │ CDC Enabled / Debezium / Azure CDC
       ▼
    Event Hub (change events)
       │
       ▼
    Auto Loader → Bronze (raw changes)
       │
       │ MERGE statement
       ▼
    Silver (current state)
```

- *"MERGE INTO target USING changes ON key WHEN MATCHED UPDATE WHEN NOT MATCHED INSERT"*

---

#### Q10: "*What SCD type would you use for customer address changes?*"

**What they're testing:** SCD pattern knowledge

**Expected answer:**
- *"Type 2 for historical tracking"*
- *"Start_date, end_date, is_current columns"*
- *"New row for each change, old row closed"*
- *"Enables point-in-time queries"*

---

### Category 5: Orchestration

---

#### Q11: "*How do multiple notebooks pass data to each other?*"

**What they're testing:** Databricks orchestration

**Expected answer:**
- *"Primary: Delta tables between notebooks"*
- *"Parameters: Widgets and dbutils.widgets"*
- *"Orchestration: Databricks Workflows or ADF"*
- *"NOT temp views (scope limited)"*

---

#### Q12: "*Databricks Workflows vs Azure Data Factory?*"

**What they're testing:** Tool selection

**Expected answer:**
- *"Workflows: Pure Databricks, simple, native"*
- *"ADF: Multi-service orchestration, more triggers"*
- *"Use Workflows if all compute is Databricks"*
- *"Use ADF if copying from multiple sources or non-Spark"*

---

### Category 6: Governance

---

#### Q13: "*How do you track data lineage?*"

**What they're testing:** Governance understanding

**Expected answer:**
- *"Unity Catalog for Databricks - automatic lineage"*
- *"Azure Purview for cross-service lineage"*
- *"Lineage: Source → Transformations → Target"*
- *"Enables impact analysis for schema changes"*

---

#### Q14: "*How do you handle PII data?*"

**What they're testing:** Security awareness

**Expected answer:**
- *"Identify PII columns upfront"*
- *"Column-level masking in Silver layer"*
- *"Row-level security for multi-tenant"*
- *"Unity Catalog ACLs for access control"*
- *"Audit logging for compliance"*

---

## 🎯 Quick Reference: Phrase Bank

Use these phrases to sound confident:

| Topic | Phrase |
|-------|--------|
| Architecture | *"I'd structure this as Bronze-Silver-Gold layers..."* |
| Scale | *"At petabyte scale, the key is incremental processing..."* |
| Failures | *"This is idempotent by design, so reruns are safe..."* |
| Quality | *"Bad records go to a quarantine table for later retry..."* |
| Streaming | *"Watermarks handle late-arriving events by..."* |
| Joins | *"For a large-small join, I'd use broadcast..."* |

---

## 📖 Study Path

1. Practice drawing the **5-layer template** from memory
2. Rehearse the **Lambda architecture** explanation
3. Walk through **Late-arriving data** scenario out loud
4. Review **all 14 questions** with timer (3 min each)
