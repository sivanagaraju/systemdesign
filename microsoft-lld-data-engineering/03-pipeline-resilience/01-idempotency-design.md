# Idempotency Design

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Critical Concept)

## The Core Question

*"If your pipeline runs twice by accident, does it duplicate data?"*

An **idempotent** pipeline produces the same result whether it runs once or multiple times.

---

## 🤔 Why Does This Matter?

**Real-world scenario:**
1. Pipeline starts processing 1M records
2. At 800K, the cluster node crashes
3. Scheduler retry kicks in
4. Pipeline restarts from the beginning
5. **Without idempotency:** 800K records now duplicated!

---

## 📊 Idempotency Patterns

### Pattern 1: Processed Files Tracking

Track which files have been processed in a metadata table.

```python
# Metadata Table Schema
class ProcessedFilesMetadata:
    file_path: str          # Primary key
    file_hash: str          # MD5 for content verification
    processed_at: datetime
    status: str             # SUCCESS, FAILED
    record_count: int
```

**Implementation:**

```python
def process_files_idempotently(spark, source_path, metadata_table):
    # Step 1: List new files
    all_files = dbutils.fs.ls(source_path)
    
    # Step 2: Get already processed files
    processed = spark.table(metadata_table).select("file_path").collect()
    processed_paths = {row.file_path for row in processed}
    
    # Step 3: Filter to only new files
    new_files = [f for f in all_files if f.path not in processed_paths]
    
    if not new_files:
        print("No new files to process")
        return
    
    # Step 4: Process each file
    for file in new_files:
        try:
            df = spark.read.parquet(file.path)
            
            # Your transformation logic
            result = transform(df)
            
            # Write to destination
            result.write.mode("append").saveAsTable("destination_table")
            
            # Step 5: Mark as processed
            spark.sql(f"""
                INSERT INTO {metadata_table}
                VALUES ('{file.path}', '{compute_hash(file)}', 
                        current_timestamp(), 'SUCCESS', {df.count()})
            """)
            
        except Exception as e:
            # Mark as failed for retry
            spark.sql(f"""
                INSERT INTO {metadata_table}
                VALUES ('{file.path}', NULL, current_timestamp(), 
                        'FAILED', 0)
            """)
            raise
```

---

### Pattern 2: Overwrite Partition

Replace entire partition on each run - naturally idempotent.

```python
def process_date_partition(spark, date_str):
    # Read source data for this date
    df = spark.read.parquet(f"/source/date={date_str}")
    
    # Transform
    result = transform(df)
    
    # Overwrite ONLY this partition
    result.write \
        .format("delta") \
        .mode("overwrite") \
        .option("replaceWhere", f"date = '{date_str}'") \
        .saveAsTable("destination_table")
    
    # If called again with same date, no duplicates - partition replaced!
```

**Key insight:** The entire partition is replaced, so re-running is safe.

---

### Pattern 3: MERGE (Upsert)

Use MERGE for record-level idempotency.

```python
from delta.tables import DeltaTable

def upsert_records(spark, updates_df):
    target = DeltaTable.forPath(spark, "/delta/destination")
    
    # MERGE: Update if exists, insert if new
    target.alias("target").merge(
        updates_df.alias("source"),
        condition="target.id = source.id"
    ).whenMatchedUpdateAll() \    # Update existing
     .whenNotMatchedInsertAll() \  # Insert new
     .execute()
    
    # Running twice with same data = no duplicates!
```

---

### Pattern 4: Insert with Deduplication

When insert-only, deduplicate before writing.

```python
def insert_with_dedup(spark, new_data_df, target_table, key_columns):
    # Get existing keys
    existing = spark.table(target_table).select(*key_columns)
    
    # Anti-join to find truly new records
    new_only = new_data_df.join(
        existing,
        on=key_columns,
        how="left_anti"  # Only rows NOT in existing
    )
    
    # Safe to insert - these are guaranteed new
    new_only.write.mode("append").saveAsTable(target_table)
```

---

## 🏗️ LLD: Idempotent Ingestion Class

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
import hashlib

class ProcessingStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    SUCCESS = "success"
    FAILED = "failed"

@dataclass
class FileMetadata:
    file_path: str
    file_hash: str
    status: ProcessingStatus
    processed_at: datetime = None
    record_count: int = 0
    error_message: str = None


class IdempotentIngestor:
    """
    Idempotent file ingestion with exactly-once processing guarantees.
    """
    
    def __init__(self, spark, metadata_table: str, destination_table: str):
        self.spark = spark
        self.metadata_table = metadata_table
        self.destination_table = destination_table
    
    def compute_file_hash(self, file_path: str) -> str:
        """Compute MD5 hash of file for content-based deduplication."""
        content = self.spark.read.format("binaryFile").load(file_path)
        # Simplified - in reality, use proper hashing
        return hashlib.md5(file_path.encode()).hexdigest()
    
    def is_already_processed(self, file_path: str, file_hash: str) -> bool:
        """Check if file was already successfully processed."""
        result = self.spark.sql(f"""
            SELECT 1 FROM {self.metadata_table}
            WHERE file_path = '{file_path}'
              AND file_hash = '{file_hash}'
              AND status = 'SUCCESS'
        """)
        return result.count() > 0
    
    def mark_processing_started(self, file_path: str, file_hash: str):
        """Mark file as in-progress (lock mechanism)."""
        self.spark.sql(f"""
            MERGE INTO {self.metadata_table} AS target
            USING (SELECT '{file_path}' as file_path) AS source
            ON target.file_path = source.file_path
            WHEN MATCHED THEN
                UPDATE SET status = 'IN_PROGRESS', processed_at = current_timestamp()
            WHEN NOT MATCHED THEN
                INSERT (file_path, file_hash, status, processed_at)
                VALUES ('{file_path}', '{file_hash}', 'IN_PROGRESS', current_timestamp())
        """)
    
    def mark_processing_complete(self, file_path: str, record_count: int):
        """Mark file as successfully processed."""
        self.spark.sql(f"""
            UPDATE {self.metadata_table}
            SET status = 'SUCCESS', record_count = {record_count}
            WHERE file_path = '{file_path}'
        """)
    
    def mark_processing_failed(self, file_path: str, error: str):
        """Mark file as failed for retry."""
        escaped_error = error.replace("'", "''")
        self.spark.sql(f"""
            UPDATE {self.metadata_table}
            SET status = 'FAILED', error_message = '{escaped_error}'
            WHERE file_path = '{file_path}'
        """)
    
    def ingest_file(self, file_path: str, transform_fn):
        """
        Idempotently ingest a single file.
        
        Args:
            file_path: Path to source file
            transform_fn: Transformation function to apply
        """
        file_hash = self.compute_file_hash(file_path)
        
        # Check idempotency - skip if already processed
        if self.is_already_processed(file_path, file_hash):
            print(f"Skipping already processed file: {file_path}")
            return
        
        # Acquire "lock" via metadata
        self.mark_processing_started(file_path, file_hash)
        
        try:
            # Read and transform
            df = self.spark.read.parquet(file_path)
            result = transform_fn(df)
            
            # Write to destination
            result.write.mode("append").saveAsTable(self.destination_table)
            
            # Mark success
            self.mark_processing_complete(file_path, result.count())
            
        except Exception as e:
            self.mark_processing_failed(file_path, str(e))
            raise
```

---

## 🎯 Interview Answer Framework

When asked about idempotency:

> **Definition:**
> *"An idempotent pipeline produces the same result whether run once or multiple times with the same input."*

> **Why it matters:**
> *"In distributed systems, retries are common due to failures. Without idempotency, retries cause data duplication."*

> **Implementation strategies:**
> 1. *"Track processed files in a metadata table - skip already processed"*
> 2. *"Use partition overwrite - replace entire partition each run"*
> 3. *"Use MERGE/upsert - update if exists, insert if new"*
> 4. *"Deduplicate before insert - anti-join with existing data"*

---

## 📖 Next Topic

Continue to [Checkpointing & Recovery](./02-checkpointing-recovery.md) to learn resumable pipelines.
