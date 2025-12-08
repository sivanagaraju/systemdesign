# Petabyte-Scale Patterns

> **When they say "we have petabytes of data"**

## The Core Question

*"We process petabytes of IoT data daily. How does your architecture handle this scale?"*

---

## 📊 Scale Considerations

| Scale | Daily Volume | Architecture Complexity |
|-------|--------------|------------------------|
| **Gigabytes** | < 100 GB | Simple, single notebook |
| **Terabytes** | 100 GB - 10 TB | Partitioning, parallel jobs |
| **Petabytes** | 10+ TB | Distributed, incremental, careful design |

---

## 🏗️ PB-Scale Architecture

```mermaid
graph LR
    subgraph Ingestion ["Ingestion & Optimization"]
        Raw[Raw Events] --> |"Auto Loader"| Bronze[Bronze Delta Table]
    end
    
    subgraph Storage_Layout ["Petabyte Storage Layout"]
        direction TB
        root["/bronze/events/"]
        
        %% Partitioning Layer
        root --> Y24["year=2024"]
        Y24 --> M01["month=01"]
        M01 --> D15["day=15"]
        
        %% File Layer with Z-Order
        D15 --> File1["part-001.parquet"]
        D15 --> File2["part-002.parquet"]
        
        style Y24 fill:#fff3e0
        style File1 fill:#e0f2f1
    end
    
    subgraph Data_Skipping ["Query: WHERE day=15 AND device_id=99"]
        Query --> |"Partition Pruning"| D15
        D15 -.-> |"Z-Order Skipping"| File2
        Note["Skips File1 entirely<br/>(Min/Max stats)"] -.-> File2
    end

    subgraph Lifecycle ["Tiering Lifecycle"]
        Hot[("Hot Tier (SSD)")] --> |"Age > 30 Days"| Warm[("Cool Tier (HDD)")]
        Warm --> |"Age > 1 Year"| Cold[("Archive (Glacier)")]
    end
    
    Bronze --> root
```

---

## 🔄 Retry Policies at Scale

```python
# Databricks Workflow retry configuration
{
    "task_key": "bronze_to_silver",
    "retry_policy": {
        "max_retries": 3,
        "min_retry_interval_millis": 60000,    # 1 min
        "max_retry_interval_millis": 300000    # 5 min (exponential backoff)
    },
    "timeout_seconds": 7200,  # 2 hours max
    "email_notifications": {
        "on_failure": ["data-team@company.com"]
    }
}
```

---

## ⚡ Key Patterns for PB Scale

| Pattern | Why | How |
|---------|-----|-----|
| **Partition pruning** | Don't scan all data | Filter on partition columns first |
| **Incremental processing** | Process only new data | Track watermark, checkpoint |
| **File compaction** | Avoid small files problem | OPTIMIZE command, Auto-Optimize |
| **Z-Order** | Skip irrelevant rowgroups | Z-ORDER BY frequently filtered columns |
| **Adaptive Query Execution** | Handle skew automatically | `spark.sql.adaptive.enabled = true` |

---

## 🎯 Interview Answer

> *"At petabyte scale, key considerations are:*
>
> 1. **Partition by time** (year/month/day/hour) for partition pruning
> 2. **Incremental processing** - never full table scans, use watermarks
> 3. **File size targets** - 256MB-1GB to avoid small files
> 4. **Autoscaling clusters** with spot instances for cost
> 5. **Retry with exponential backoff** for transient failures
> 6. **AQE enabled** for automatic skew handling"*

---

## 📖 Next Topic

Continue to [Realistic Interview Questions](./06-realistic-interview-questions.md) for practice.
