# Dead Letter Queues (DLQ)

> **Interview Frequency:** ⭐⭐⭐⭐ (Data Quality Question)

## The Core Question

*"Design a mechanism to block 'bad data' from entering the Silver layer without stopping the pipeline."*

---

## 🤔 The Problem

```
Raw Data (1M records)
        │
        ▼
   ┌─────────────────────────────┐
   │     Data Validation         │
   │  ┌───────────────────────┐  │
   │  │ 990,000 valid records │──┼──► Silver Layer ✓
   │  └───────────────────────┘  │
   │  ┌───────────────────────┐  │
   │  │ 10,000 invalid records│──┼──► ??? 
   │  └───────────────────────┘  │
   └─────────────────────────────┘
```

**Options for invalid records:**
1. ❌ **Fail the entire pipeline** - Bad: 990K good records lost
2. ❌ **Drop silently** - Bad: Data loss, no visibility
3. ✅ **Route to Dead Letter Queue** - Good: Pipeline continues, bad data preserved

---

## 🗂️ DLQ Design

### Quarantine Table Schema

```sql
CREATE TABLE quarantine_records (
    -- Original record
    raw_record STRING,             -- JSON/CSV of original
    
    -- Error context
    error_type STRING,             -- 'VALIDATION', 'SCHEMA', 'BUSINESS_RULE'
    error_code STRING,             -- 'NULL_REQUIRED_FIELD', 'INVALID_DATE'
    error_message STRING,          -- Human readable
    error_details STRING,          -- JSON with failed validations
    
    -- Metadata
    source_file STRING,            -- Where it came from
    source_system STRING,          -- Which upstream system
    record_id STRING,              -- If identifiable
    
    -- Timestamps
    ingested_at TIMESTAMP,         -- When originally received
    quarantined_at TIMESTAMP,      -- When moved to DLQ
    
    -- Reprocessing
    retry_count INT DEFAULT 0,     -- Number of retry attempts  
    last_retry_at TIMESTAMP,
    status STRING DEFAULT 'PENDING' -- PENDING, RESOLVED, ABANDONED
)
PARTITIONED BY (quarantined_date DATE, error_type STRING);
```

---

## 🏗️ Implementation: Validator Class

```python
from dataclasses import dataclass
from typing import List, Tuple, Callable
from pyspark.sql import DataFrame
from pyspark.sql.functions import col, when, lit, to_json, struct, current_timestamp

@dataclass
class ValidationRule:
    """Single validation rule."""
    name: str
    condition: str           # SQL expression that returns TRUE for VALID records
    error_code: str
    error_message: str


class DataValidator:
    """
    Validates data and routes failures to DLQ while passing valid records through.
    """
    
    def __init__(self, spark, quarantine_table: str):
        self.spark = spark
        self.quarantine_table = quarantine_table
        self.rules: List[ValidationRule] = []
    
    def add_rule(self, name: str, condition: str, error_code: str, error_message: str):
        """Add a validation rule."""
        self.rules.append(ValidationRule(name, condition, error_code, error_message))
        return self  # For chaining
    
    def validate(self, df: DataFrame, source_file: str = "unknown") -> Tuple[DataFrame, DataFrame]:
        """
        Validate DataFrame and split into valid/invalid.
        
        Returns:
            Tuple of (valid_df, invalid_df)
        """
        # Build combined validation expression
        # A record is valid if ALL rules pass
        all_valid_condition = None
        
        for rule in self.rules:
            rule_result = col(rule.condition) if isinstance(rule.condition, str) else rule.condition
            if all_valid_condition is None:
                all_valid_condition = rule_result
            else:
                all_valid_condition = all_valid_condition & rule_result
        
        # Add validation flag
        validated = df.withColumn("_is_valid", all_valid_condition)
        
        # Split into valid and invalid
        valid_df = validated.filter(col("_is_valid")).drop("_is_valid")
        invalid_df = validated.filter(~col("_is_valid")).drop("_is_valid")
        
        return valid_df, invalid_df
    
    def process_with_dlq(self, df: DataFrame, source_file: str) -> DataFrame:
        """
        Validate data, write invalid to DLQ, return valid.
        
        Args:
            df: Input DataFrame
            source_file: Source file path for tracking
            
        Returns:
            Valid records only
        """
        valid_df, invalid_df = self.validate(df, source_file)
        
        if invalid_df.count() > 0:
            # Prepare for quarantine
            quarantine_df = invalid_df.select(
                to_json(struct("*")).alias("raw_record"),
                lit("VALIDATION").alias("error_type"),
                lit(self._get_first_failing_rule(invalid_df)).alias("error_code"),
                lit("Validation failed").alias("error_message"),
                lit(source_file).alias("source_file"),
                current_timestamp().alias("quarantined_at"),
                lit("PENDING").alias("status")
            )
            
            # Write to quarantine table
            quarantine_df.write \
                .mode("append") \
                .saveAsTable(self.quarantine_table)
            
            print(f"Quarantined {invalid_df.count()} invalid records")
        
        return valid_df
    
    def _get_first_failing_rule(self, df: DataFrame) -> str:
        """Get the first rule that failed for error reporting."""
        # Simplified - in reality, check each rule
        return self.rules[0].error_code if self.rules else "UNKNOWN"


# Usage Example
validator = DataValidator(spark, "data_quarantine")

validator \
    .add_rule(
        name="required_customer_id",
        condition="customer_id IS NOT NULL AND customer_id != ''",
        error_code="NULL_CUSTOMER_ID",
        error_message="Customer ID is required"
    ) \
    .add_rule(
        name="valid_amount",
        condition="amount >= 0",
        error_code="NEGATIVE_AMOUNT",
        error_message="Amount cannot be negative"
    ) \
    .add_rule(
        name="valid_date",
        condition="order_date <= current_date()",
        error_code="FUTURE_DATE",
        error_message="Order date cannot be in the future"
    )

# Process data
valid_records = validator.process_with_dlq(raw_df, "/source/orders_2024.csv")

# Continue pipeline with valid records only
valid_records.write.mode("append").saveAsTable("silver.orders")
```

---

## 🔄 DLQ Reprocessing

### Manual Reprocessing

```sql
-- View quarantined records by error type
SELECT error_code, COUNT(*) as count
FROM quarantine_records
WHERE status = 'PENDING'
GROUP BY error_code
ORDER BY count DESC;

-- Fix and reprocess specific error type
-- After fixing upstream issue:
UPDATE quarantine_records
SET status = 'REPROCESSING'
WHERE error_code = 'INVALID_DATE' 
  AND status = 'PENDING';
```

### Automated Reprocessing Job

```python
class DLQReprocessor:
    """Automatic retry of quarantined records."""
    
    def __init__(self, spark, quarantine_table: str, max_retries: int = 3):
        self.spark = spark
        self.quarantine_table = quarantine_table
        self.max_retries = max_retries
    
    def reprocess_pending(self, validator, target_table: str):
        """Attempt to reprocess pending quarantine records."""
        
        # Get retry-eligible records
        pending = self.spark.sql(f"""
            SELECT * FROM {self.quarantine_table}
            WHERE status = 'PENDING'
              AND retry_count < {self.max_retries}
              AND (last_retry_at IS NULL OR 
                   last_retry_at < current_timestamp() - INTERVAL 1 HOUR)
        """)
        
        if pending.count() == 0:
            print("No records to reprocess")
            return
        
        # Parse raw records back to DataFrame
        from pyspark.sql.functions import from_json, schema_of_json
        
        sample_record = pending.select("raw_record").first()[0]
        schema = schema_of_json(sample_record)
        
        records_df = pending.select(
            from_json("raw_record", schema).alias("data"),
            "record_id"
        ).select("data.*", "record_id")
        
        # Re-validate
        valid_df, still_invalid = validator.validate(records_df)
        
        # Write valid to target
        if valid_df.count() > 0:
            valid_df.write.mode("append").saveAsTable(target_table)
            
            # Mark as resolved
            # (simplified - use join with record_id)
        
        # Update retry count for still-invalid
        if still_invalid.count() > 0:
            # Update retry_count and last_retry_at
            pass
```

---

## 🎯 Interview Answer Framework

> *"For bad data handling, I'd implement a Dead Letter Queue pattern:*
>
> 1. **Validation Layer:**
>    - Define validation rules (nulls, types, business rules)
>    - Split data into valid/invalid streams
>
> 2. **Quarantine Table:**
>    - Store invalid records with error details
>    - Include: raw record, error type, source file, timestamp
>    - Partition by date and error type for efficient querying
>
> 3. **Pipeline Continues:**
>    - Valid records proceed to Silver layer
>    - Invalid records written to quarantine
>    - No pipeline failures for data quality issues
>
> 4. **Reprocessing:**
>    - Daily job retries eligible quarantine records
>    - After N failures, mark as 'ABANDONED' for manual review
>    - Track retry count and last retry timestamp"

---

## 📖 Next Topic

Continue to [Azure Data Factory Deep Dive](./04-azure-data-factory-deep-dive.md) for ADF orchestration patterns.
