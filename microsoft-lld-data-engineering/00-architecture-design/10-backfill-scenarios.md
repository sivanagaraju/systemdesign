# Backfill Scenarios

> **Reprocessing historical data at scale**

## The Core Problem

*"We found a bug in our transformation. We need to reprocess the last 6 months of data. How do you design for this?"*

```
Normal Daily Pipeline:         Backfill Requirement:
───────────────────────        ───────────────────────
Process today's data           Reprocess 6 months
~10 GB                         ~1.8 TB (180 days × 10GB)

If you run same pipeline → 
- Takes 180x longer
- May timeout
- Overwhelms cluster
- Blocks daily runs
```

---

## 🏗️ Backfill Architecture

```mermaid
flowchart TD
    %% Partition-wise Backfill Strategy
    Start([Start Backfill Job]) --> ListPart{List Date Partitions}
    ListPart --> |"Range: Jan 1 - Jun 30"| Queue[Pending Queue]
    
    Queue --> Pick[Pick Next Partition]
    Pick --> CheckState{Already in Progress Table?}
    
    CheckState -- "Completed" --> Pick
    CheckState -- "Running" --> Skip[Skip / Wait] --> Pick
    CheckState -- "Pending" --> Process[Process Partition]
    
    subgraph Execution ["Spark Execution"]
        Process --> Read[Read Raw Data (Pruned)]
        Read --> Transform[Apply Logic]
        Transform --> Write{Write Success?}
    end
    
    Write -- "Yes" --> Log[Update Progress: COMPLETE] --> Pick
    Write -- "No" --> Retry{Retry Count < 3?}
    
    Retry -- "Yes" --> Process
    Retry -- "No" --> Alert[Log Error & Continue] --> Pick
    
    Pick -- "Queue Empty" --> End([Backfill Complete])
    
    style Process fill:#e1f5fe
    style Log fill:#c8e6c9
    style Alert fill:#ffcdd2
```

---

## 🔧 Code Implementation

### Backfill Controller

```python
from datetime import datetime, timedelta
from pyspark.sql.functions import *

class BackfillController:
    """
    Manages partition-by-partition backfill with progress tracking.
    """
    
    def __init__(self, spark, progress_table: str):
        self.spark = spark
        self.progress_table = progress_table
        self._ensure_progress_table()
    
    def _ensure_progress_table(self):
        self.spark.sql(f"""
            CREATE TABLE IF NOT EXISTS {self.progress_table} (
                partition_date DATE,
                status STRING,
                started_at TIMESTAMP,
                completed_at TIMESTAMP,
                error_message STRING
            ) USING DELTA
        """)
    
    def get_pending_partitions(self, start_date: str, end_date: str) -> list:
        """Get partitions that haven't been processed yet."""
        
        # Get already completed partitions
        completed = self.spark.sql(f"""
            SELECT partition_date FROM {self.progress_table}
            WHERE status = 'COMPLETE'
        """).collect()
        completed_dates = {row.partition_date for row in completed}
        
        # Generate all dates in range
        all_dates = []
        current = datetime.strptime(start_date, "%Y-%m-%d")
        end = datetime.strptime(end_date, "%Y-%m-%d")
        while current <= end:
            if current.date() not in completed_dates:
                all_dates.append(current.strftime("%Y-%m-%d"))
            current += timedelta(days=1)
        
        return all_dates
    
    def mark_started(self, partition_date: str):
        self.spark.sql(f"""
            MERGE INTO {self.progress_table} AS target
            USING (SELECT '{partition_date}'::DATE as partition_date) AS source
            ON target.partition_date = source.partition_date
            WHEN MATCHED THEN UPDATE SET status = 'RUNNING', started_at = current_timestamp()
            WHEN NOT MATCHED THEN INSERT (partition_date, status, started_at)
                VALUES (source.partition_date, 'RUNNING', current_timestamp())
        """)
    
    def mark_complete(self, partition_date: str):
        self.spark.sql(f"""
            UPDATE {self.progress_table}
            SET status = 'COMPLETE', completed_at = current_timestamp()
            WHERE partition_date = '{partition_date}'
        """)
    
    def mark_failed(self, partition_date: str, error: str):
        escaped_error = error.replace("'", "''")
        self.spark.sql(f"""
            UPDATE {self.progress_table}
            SET status = 'FAILED', error_message = '{escaped_error}'
            WHERE partition_date = '{partition_date}'
        """)
    
    def run_backfill(self, start_date: str, end_date: str, process_fn):
        """Run backfill for date range."""
        
        pending = self.get_pending_partitions(start_date, end_date)
        print(f"Backfill: {len(pending)} partitions to process")
        
        for partition_date in pending:
            print(f"Processing {partition_date}...")
            self.mark_started(partition_date)
            
            try:
                process_fn(partition_date)
                self.mark_complete(partition_date)
                print(f"✅ {partition_date} complete")
            except Exception as e:
                self.mark_failed(partition_date, str(e))
                print(f"❌ {partition_date} failed: {e}")
                # Continue to next partition, don't stop entire backfill


# Usage
def process_partition(partition_date: str):
    """Process a single partition - your transformation logic."""
    
    # Read only this partition
    df = spark.read.format("delta").load("/bronze/orders") \
        .filter(f"order_date = '{partition_date}'")
    
    # Transform
    result = df.withColumn("amount_usd", col("amount") * 1.1)  # Fixed bug
    
    # Overwrite only this partition (idempotent!)
    result.write.format("delta") \
        .mode("overwrite") \
        .option("replaceWhere", f"order_date = '{partition_date}'") \
        .save("/silver/orders")


# Run backfill
controller = BackfillController(spark, "backfill_progress")
controller.run_backfill("2024-01-01", "2024-06-30", process_partition)
```

### Parallel Backfill with Databricks Workflows

```python
# Use Databricks Workflows to run partitions in parallel
# Each task processes different date range

# Task 1: Jan-Feb
dbutils.widgets.text("start_date", "2024-01-01")
dbutils.widgets.text("end_date", "2024-02-28")

# Task 2: Mar-Apr (runs in parallel)
# Task 3: May-Jun (runs in parallel)

# Each task:
start = dbutils.widgets.get("start_date")
end = dbutils.widgets.get("end_date")
controller = BackfillController(spark, "backfill_progress")
controller.run_backfill(start, end, process_partition)
```

---

## ⚖️ Backfill vs Daily Pipeline

| Aspect | Daily Pipeline | Backfill |
|--------|---------------|----------|
| Scope | Today's data | Months of data |
| Parallelism | Single day | Multiple partitions |
| Priority | High (SLA) | Lower (can pause) |
| Cluster | Shared | Dedicated or separate |
| Monitoring | Alerts on failure | Progress dashboard |

---

## 🎯 Interview Questions

| Question | Expected Answer |
|----------|----------------|
| *"How do you reprocess 6 months?"* | Partition by partition with progress tracking, not all at once |
| *"What if backfill fails at 50%?"* | Progress table tracks completion, resume from last PENDING |
| *"How do you not block daily runs?"* | Separate cluster or time-boxed (run nights only) |
| *"How is it idempotent?"* | `replaceWhere` overwrites entire partition - safe to re-run |

---

## 📖 Next Scenario

Continue to [CDC Deletes Handling](./11-cdc-deletes.md).
