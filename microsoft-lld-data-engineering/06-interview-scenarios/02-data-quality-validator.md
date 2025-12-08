# Scenario: Data Quality Validator

## 📝 Problem Statement

> *"Design a mechanism to block 'bad data' from entering the Silver layer without stopping the pipeline."*

**Context:** Bronze-to-Silver transformation that must be fault-tolerant.

---

## 🎯 Model Answer

### 1. Clarify

*"What types of validation? Schema, nulls, business rules?"*
*"Should bad records be recoverable for reprocessing?"*

### 2. High-Level Design

```
Bronze Data (1M records)
        │
        ▼
   ┌─────────────────┐
   │   Validator     │
   │   ┌───────────┐ │
   │   │ Valid     │─┼──► Silver Layer
   │   └───────────┘ │
   │   ┌───────────┐ │
   │   │ Invalid   │─┼──► Quarantine Table (DLQ)
   │   └───────────┘ │
   └─────────────────┘
```

### 3. Implementation

```python
class DataValidator:
    def __init__(self, spark, quarantine_table: str):
        self.spark = spark
        self.quarantine_table = quarantine_table
        self.rules = []
    
    def add_rule(self, condition: str, error_code: str):
        self.rules.append((condition, error_code))
        return self
    
    def validate(self, df):
        # Build validation condition
        valid_condition = reduce(lambda a, b: a & b,
            [expr(rule[0]) for rule in self.rules])
        
        # Split data
        valid = df.filter(valid_condition)
        invalid = df.filter(~valid_condition)
        
        # Quarantine bad records
        if invalid.count() > 0:
            invalid.withColumn("error_reason", lit("Validation failed")) \
                .write.mode("append").saveAsTable(self.quarantine_table)
        
        return valid

# Usage
validator = DataValidator(spark, "data_quarantine")
validator.add_rule("customer_id IS NOT NULL", "NULL_CUSTOMER")
validator.add_rule("amount >= 0", "NEGATIVE_AMOUNT")

valid_df = validator.validate(bronze_df)
valid_df.write.saveAsTable("silver.orders")
```

### 4. Quarantine Table Design

```sql
CREATE TABLE quarantine (
    raw_record STRING,       -- Original as JSON
    error_code STRING,
    error_message STRING,
    source_file STRING,
    quarantined_at TIMESTAMP,
    retry_count INT DEFAULT 0,
    status STRING DEFAULT 'PENDING'
) PARTITIONED BY (date, error_code);
```

---

## 📖 Next Scenario

Continue to [Streaming Late Events](./03-streaming-late-events.md).
