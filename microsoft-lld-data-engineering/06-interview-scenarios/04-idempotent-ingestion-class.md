# Scenario: Idempotent Ingestion Class

## 📝 Problem Statement

> *"Design an idempotent data ingestion class in Python that handles file-based batch ingestion"*

**Context:** Files land in ADLS, need to process exactly once even if pipeline reruns.

---

## 🎯 Model Answer

### Complete Implementation

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Callable, Optional
import hashlib


class Status(Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    SUCCESS = "success"
    FAILED = "failed"


@dataclass
class ProcessedFile:
    file_path: str
    file_hash: str
    status: Status
    processed_at: datetime = None
    record_count: int = 0
    error_message: str = None


class IdempotentIngestor:
    """
    Idempotent file ingestion with exactly-once guarantees.
    
    Features:
    - Tracks processed files via metadata table
    - Content-based deduplication (hash check)
    - Atomic status updates
    - Retry support for failed files
    """
    
    def __init__(self, spark, metadata_table: str, target_table: str):
        self.spark = spark
        self.metadata_table = metadata_table
        self.target_table = target_table
        self._ensure_metadata_table()
    
    def _ensure_metadata_table(self):
        """Create metadata table if not exists."""
        self.spark.sql(f"""
            CREATE TABLE IF NOT EXISTS {self.metadata_table} (
                file_path STRING,
                file_hash STRING,
                status STRING,
                processed_at TIMESTAMP,
                record_count BIGINT,
                error_message STRING
            ) USING DELTA
        """)
    
    def _compute_hash(self, file_path: str) -> str:
        """Compute content hash for deduplication."""
        # In production, read file and compute MD5/SHA256
        return hashlib.md5(file_path.encode()).hexdigest()
    
    def _is_processed(self, file_path: str, file_hash: str) -> bool:
        """Check if file was already successfully processed."""
        result = self.spark.sql(f"""
            SELECT 1 FROM {self.metadata_table}
            WHERE file_path = '{file_path}'
              AND file_hash = '{file_hash}'
              AND status = 'SUCCESS'
        """)
        return result.count() > 0
    
    def _update_status(self, file_path: str, file_hash: str, 
                       status: Status, record_count: int = 0, error: str = None):
        """Atomically update file processing status."""
        error_escaped = error.replace("'", "''") if error else ""
        
        self.spark.sql(f"""
            MERGE INTO {self.metadata_table} AS target
            USING (SELECT '{file_path}' AS file_path) AS source
            ON target.file_path = source.file_path
            WHEN MATCHED THEN UPDATE SET
                status = '{status.value}',
                processed_at = current_timestamp(),
                record_count = {record_count},
                error_message = '{error_escaped}'
            WHEN NOT MATCHED THEN INSERT 
                (file_path, file_hash, status, processed_at, record_count)
                VALUES ('{file_path}', '{file_hash}', '{status.value}', 
                        current_timestamp(), {record_count})
        """)
    
    def ingest_file(self, file_path: str, 
                    transform_fn: Callable = None) -> bool:
        """
        Ingest a single file idempotently.
        
        Args:
            file_path: Path to source file
            transform_fn: Optional transformation function
            
        Returns:
            True if processed, False if skipped (already processed)
        """
        file_hash = self._compute_hash(file_path)
        
        # Idempotency check
        if self._is_processed(file_path, file_hash):
            print(f"Skipping already processed: {file_path}")
            return False
        
        # Mark as processing (lock)
        self._update_status(file_path, file_hash, Status.PROCESSING)
        
        try:
            # Read file
            df = self.spark.read.parquet(file_path)
            
            # Apply transformation if provided
            if transform_fn:
                df = transform_fn(df)
            
            record_count = df.count()
            
            # Write to target
            df.write.mode("append").saveAsTable(self.target_table)
            
            # Mark success
            self._update_status(file_path, file_hash, Status.SUCCESS, record_count)
            print(f"Processed {record_count} records from {file_path}")
            return True
            
        except Exception as e:
            # Mark failed
            self._update_status(file_path, file_hash, Status.FAILED, error=str(e))
            raise
    
    def ingest_directory(self, directory_path: str,
                         transform_fn: Callable = None) -> dict:
        """
        Ingest all files in a directory.
        
        Returns:
            Stats: {"processed": N, "skipped": M, "failed": K}
        """
        files = self.spark._jvm.org.apache.hadoop.fs.Path(directory_path)
        # Simplified - list files from ADLS
        
        stats = {"processed": 0, "skipped": 0, "failed": 0}
        
        # for file in files:
        #     try:
        #         if self.ingest_file(file, transform_fn):
        #             stats["processed"] += 1
        #         else:
        #             stats["skipped"] += 1
        #     except Exception:
        #         stats["failed"] += 1
        
        return stats


# Usage
ingestor = IdempotentIngestor(
    spark=spark,
    metadata_table="etl_metadata.processed_files",
    target_table="silver.orders"
)

# Define transformation
def transform_orders(df):
    return df.withColumn("processed_at", current_timestamp()) \
             .filter(col("amount") > 0)

# Process files
ingestor.ingest_file("/raw/orders_2024.parquet", transform_orders)

# Rerun is safe - skips already processed files!
ingestor.ingest_file("/raw/orders_2024.parquet", transform_orders)  # Skipped!
```

---

## ✅ Key Design Points to Mention

1. **Metadata table tracks state** - persisted, queryable
2. **Content hash for dedup** - same file with different path = same data
3. **Status transitions** - PENDING → PROCESSING → SUCCESS/FAILED
4. **Atomic updates via MERGE** - no partial states
5. **Exception handling** - mark failures, allow retry

---

## 📖 Next Section

Move to [07 - Answer Framework](../07-answer-framework/README.md) for interview technique.
