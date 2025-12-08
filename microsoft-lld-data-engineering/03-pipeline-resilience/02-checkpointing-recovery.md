# Checkpointing & Recovery

> **Interview Frequency:** ⭐⭐⭐⭐ (Common Scenario)

## The Core Question

*"If a 100GB load fails at 90%, how do you design it to resume rather than restart?"*

---

## 🤔 Resume vs Restart

| Approach | On Failure | Cost | Use Case |
|----------|------------|------|----------|
| **Restart** | Start from beginning | High (re-read all data) | Small data, simple logic |
| **Resume** | Continue from checkpoint | Low (read only remaining) | Large data, streaming |

---

## 📊 Checkpointing Patterns

### Pattern 1: Offset-Based Checkpointing

Track position in source data, resume from that position.

```python
class OffsetCheckpoint:
    """Track processing offset for resumable ingestion."""
    
    def __init__(self, spark, checkpoint_table: str, job_name: str):
        self.spark = spark
        self.checkpoint_table = checkpoint_table
        self.job_name = job_name
    
    def get_last_offset(self) -> int:
        """Get the last successfully processed offset."""
        result = self.spark.sql(f"""
            SELECT max(offset) as last_offset
            FROM {self.checkpoint_table}
            WHERE job_name = '{self.job_name}'
              AND status = 'SUCCESS'
        """).collect()[0]
        
        return result.last_offset if result.last_offset else 0
    
    def save_offset(self, offset: int, records_processed: int):
        """Save checkpoint after successful batch processing."""
        self.spark.sql(f"""
            INSERT INTO {self.checkpoint_table}
            VALUES ('{self.job_name}', {offset}, {records_processed}, 
                    'SUCCESS', current_timestamp())
        """)


def resumable_batch_process(spark, source_df, batch_size=10000):
    """Process data in batches with checkpointing."""
    
    checkpoint = OffsetCheckpoint(spark, "checkpoints", "my_job")
    last_offset = checkpoint.get_last_offset()
    
    # Resume from last successful offset
    remaining_data = source_df.filter(f"offset > {last_offset}")
    
    total_rows = remaining_data.count()
    processed = 0
    
    while processed < total_rows:
        # Process one batch
        batch = remaining_data \
            .filter(f"offset > {last_offset + processed}") \
            .limit(batch_size)
        
        batch_count = batch.count()
        if batch_count == 0:
            break
        
        # Do processing
        result = transform(batch)
        result.write.mode("append").saveAsTable("destination")
        
        # Checkpoint after each successful batch
        processed += batch_count
        checkpoint.save_offset(last_offset + processed, batch_count)
        
        print(f"Processed {processed}/{total_rows}")
```

---

### Pattern 2: Structured Streaming Checkpoints

Spark Structured Streaming has built-in checkpointing.

```python
# Streaming query with checkpoint
query = (
    spark.readStream
        .format("delta")
        .load("/source/events")
    
    .writeStream
        .format("delta")
        .outputMode("append")
        .option("checkpointLocation", "/checkpoints/my_stream")  # Checkpoint!
        .trigger(processingTime="10 seconds")
        .start("/destination/events")
)

# On failure, Spark automatically resumes from checkpoint!
```

**What's in the checkpoint directory:**

```
/checkpoints/my_stream/
├── commits/           # Completed batch IDs
│   ├── 0
│   ├── 1
│   └── 2
├── offsets/           # Source offsets per batch
│   ├── 0
│   ├── 1
│   └── 2
├── sources/           # Source metadata
│   └── 0/
│       └── 0
└── state/             # Aggregation state (for stateful queries)
    └── 0/
```

---

### Pattern 3: File-Based Checkpointing

For batch jobs, track processed files in a simple file.

```python
import json
from pathlib import Path

class FileCheckpointer:
    def __init__(self, checkpoint_path: str):
        self.checkpoint_path = checkpoint_path
    
    def load_checkpoint(self) -> dict:
        """Load checkpoint from file."""
        path = Path(self.checkpoint_path)
        if path.exists():
            return json.loads(path.read_text())
        return {"last_file": None, "last_offset": 0, "processed_files": []}
    
    def save_checkpoint(self, state: dict):
        """Save checkpoint to file."""
        Path(self.checkpoint_path).write_text(json.dumps(state, indent=2))
    
    def is_file_processed(self, file_path: str) -> bool:
        """Check if file already processed."""
        state = self.load_checkpoint()
        return file_path in state.get("processed_files", [])
    
    def mark_file_processed(self, file_path: str):
        """Mark file as processed."""
        state = self.load_checkpoint()
        state["processed_files"].append(file_path)
        state["last_file"] = file_path
        self.save_checkpoint(state)
```

---

### Pattern 4: Transaction Log Approach (Delta Lake)

Delta Lake's transaction log provides natural checkpointing.

```python
# Delta automatically tracks what's been written
df.write \
    .format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .save("/delta/destination")

# If job fails mid-write, Delta's atomicity ensures:
# - Complete write committed, OR
# - No partial data written
# Next run simply continues appending new data
```

---

## 🔧 Implementing Resume for Large File

**Scenario:** Loading a 100GB CSV that fails at 90%

```python
class ResumableFileLoader:
    """Load large files with resume capability."""
    
    def __init__(self, spark, checkpoint_path: str):
        self.spark = spark
        self.checkpoint_path = checkpoint_path
    
    def load_file(self, file_path: str, destination: str, chunk_size_mb: int = 128):
        """Load large file in chunks with checkpointing."""
        
        # Get file info
        file_size = dbutils.fs.ls(file_path)[0].size
        
        # Load checkpoint
        checkpoint = self._load_checkpoint()
        start_byte = checkpoint.get("bytes_processed", 0)
        
        if start_byte >= file_size:
            print("File already fully processed")
            return
        
        print(f"Resuming from byte {start_byte}/{file_size}")
        
        # For CSV: we need row-based checkpointing
        # Read from start, skip already processed rows
        df = self.spark.read.csv(file_path)
        
        rows_to_skip = checkpoint.get("rows_processed", 0)
        remaining = df.limit(1000000).tail(1000000 - rows_to_skip)  # Simplified
        
        # Process in batches
        # ... (batch processing logic with checkpointing)
```

---

## ⚠️ Checkpoint Gotchas

| Gotcha | Problem | Solution |
|--------|---------|----------|
| **Stale checkpoints** | Code changed but checkpoint refers to old schema | Version checkpoints, clear on schema change |
| **Large state** | Streaming checkpoint grows unbounded | Set watermark, configure state TTL |
| **Concurrent writes** | Two jobs update same checkpoint | Use atomic writes, distributed locking |
| **Checkpoint location** | Lost checkpoint = restart from scratch | Store in reliable storage (ADLS, S3) |

---

## 🎯 Interview Answer Framework

> *"For resume capability, I'd implement checkpointing:*
>
> 1. *For streaming: Use Spark's built-in checkpoint location*
> 2. *For batch: Track progress in a metadata table with offset tracking*
> 3. *After each successful batch, save the offset*
> 4. *On startup, read last offset and continue from there*
>
> *Key considerations:*
> - *Checkpoints must be atomic (transaction or overwrite)*
> - *Store in reliable storage (ADLS, not local disk)*
> - *Handle schema evolution (version checkpoints)*"

---

## 📖 Next Topic

Continue to [Dead Letter Queues](./03-dead-letter-queues.md) to learn error handling patterns.
