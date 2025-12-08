# Scenario: Streaming Late Events

## 📝 Problem Statement

> *"How do you handle late-arriving events in an IoT stream?"*

**Context:** Device telemetry where events may arrive minutes or hours late.

---

## 🎯 Model Answer

### 1. The Problem

```
Event Time: 10:00  10:05  10:10  10:15  10:20
            ──────────────────────────────────►
            
Events arrive out of order:
- Event A (10:00) arrives at 10:02 ✓
- Event B (10:05) arrives at 10:07 ✓  
- Event C (10:02) arrives at 10:15 ⚠️ Late!

If window [10:00-10:10] already closed, Event C is dropped!
```

### 2. Solution: Watermarking

*"Watermarking defines how long to wait for late data"*

```python
from pyspark.sql.functions import window

streaming_df = spark.readStream \
    .format("kafka") \
    .option("subscribe", "iot-events") \
    .load()

# Allow 10 minutes of lateness
result = streaming_df \
    .withWatermark("event_time", "10 minutes") \
    .groupBy(
        window("event_time", "5 minutes"),
        "device_id"
    ) \
    .agg(avg("temperature"), count("*"))

result.writeStream \
    .outputMode("append") \
    .format("delta") \
    .start()
```

### 3. How Watermark Works

```
Stream Progress:
─────────────────────────────────────────────►
|    10:00    |    10:10    |    10:20    |
              │             │
              │ Watermark   │
              │ = Max Event │
              │   Time - 10m│
              │             │
At 10:20, watermark = 10:10
Window [10:00-10:05] is complete → emit results
Events with event_time < 10:10 are dropped
```

### 4. Trade-offs

| Approach | Latency | Completeness |
|----------|---------|--------------|
| Small watermark (1 min) | Low | Miss more late events |
| Large watermark (1 hour) | High | Catch more late events |

---

## 📖 Next Scenario

Continue to [Idempotent Ingestion Class](./04-idempotent-ingestion-class.md).
