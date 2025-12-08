# Schema Drift Handling

> **When columns change unexpectedly**

## The Core Problem

*"A new column 'discount_code' suddenly appears in source data. Your pipeline breaks. How do you handle this?"*

```
Day 1 Schema:                      Day 2 Schema (NEW COLUMN!):
┌────────────────────────────┐     ┌────────────────────────────┐
│ order_id: string           │     │ order_id: string           │
│ customer_id: string        │     │ customer_id: string        │
│ amount: double             │     │ amount: double             │
│ order_date: date           │     │ order_date: date           │
└────────────────────────────┘     │ discount_code: string  ← NEW!
                                   └────────────────────────────┘

Without handling: Pipeline FAILS!
"AnalysisException: Cannot write to Delta table with different schema"
```

---

## 🏗️ Schema Drift Architecture

```mermaid
flowchart TD
    %% Decision Flow for Schema Drift
    NewData[Incoming Micro-Batch] --> Check{Schema Match?}
    
    Check -- "Yes" --> Write[Write to Delta]
    Check -- "No (Drift Detected)" --> NewCol{Is it a New Column?}
    
    %% Scenario 1: New Column (Additive)
    NewCol -- "Yes" --> AllowEvol{Evolution Enabled?}
    AllowEvol -- "Yes (mergeSchema)" --> Evolve[Update Table Schema] 
    Evolve --> Write
    
    AllowEvol -- "No" --> Fail[❌ Fail Job]
    
    %% Scenario 2: Incompatible Change
    NewCol -- "No (Type Mismatch / Rename)" --> Compat{Safely Castable?}
    Compat -- "Yes (Upcast)" --> Evolve
    Compat -- "No (Data Loss Risk)" --> DLQ[("⚠️ Quarantine to DLQ")]
    
    style Fail fill:#ffcdd2,stroke:#c62828
    style DLQ fill:#fff9c4,stroke:#fbc02d
    style Write fill:#c8e6c9,stroke:#2e7d32
```│                                      │                                       │
│                                      ▼                                       │
│  SCHEMA REGISTRY (Governance)                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  Unity Catalog / Schema Registry                                         ││
│  │                                                                          ││
│  │  ┌───────────────────────────────────────────────────────────────────┐  ││
│  │  │ Version 1: {order_id, amount}                                      │  ││
│  │  │ Version 2: {order_id, amount, discount_code} ← Compatible          │  ││
│  │  │ Version 3: {order_id, total}                 ← BREAKING (renamed)  │  ││
│  │  └───────────────────────────────────────────────────────────────────┘  ││
│  │                                                                          ││
│  │  Compatibility Mode:                                                     ││
│  │  - BACKWARD: New schema can read old data ← Most common                 ││
│  │  - FORWARD: Old schema can read new data                                 ││
│  │  - FULL: Both directions                                                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                      │                                       │
│                                      ▼                                       │
│  SILVER LAYER (Schema Enforcement)                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  Option 1: Strict Schema                                                 ││
│  │  ┌───────────────────────────────────────────────────────────────────┐  ││
│  │  │ df.write.format("delta")                                           │  ││
│  │  │   .mode("append")                                                  │  ││
│  │  │   .option("mergeSchema", "false")  ← REJECT schema changes        │  ││
│  │  │   .save("/silver/orders")                                          │  ││
│  │  └───────────────────────────────────────────────────────────────────┘  ││
│  │                                                                          ││
│  │  Option 2: Explicit Column Selection                                     ││
│  │  ┌───────────────────────────────────────────────────────────────────┐  ││
│  │  │ expected_columns = ["order_id", "amount", "order_date"]            │  ││
│  │  │ df.select([col(c) for c in expected_columns])  ← Ignore new cols  │  ││
│  │  └───────────────────────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Code Implementation

### Auto Loader with Schema Evolution

```python
# Bronze: Accept all schema changes
bronze_stream = spark.readStream.format("cloudFiles") \
    .option("cloudFiles.format", "json") \
    .option("cloudFiles.schemaLocation", "/schemas/orders") \
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns") \
    .option("cloudFiles.inferColumnTypes", "true") \
    .load("/raw/orders/")

# Write to Bronze with schema merge
bronze_stream.writeStream \
    .format("delta") \
    .option("mergeSchema", "true") \
    .option("checkpointLocation", "/checkpoints/bronze_orders") \
    .start("/bronze/orders")
```

### Silver: Explicit Column Selection

```python
# Define expected schema (contract)
EXPECTED_COLUMNS = [
    "order_id",
    "customer_id", 
    "amount",
    "order_date"
]

# Read bronze (may have extra columns)
bronze_df = spark.read.format("delta").load("/bronze/orders")

# Select only expected columns (ignore new ones)
# Handle missing columns with lit(None)
def safe_select(df, columns):
    result_cols = []
    for col_name in columns:
        if col_name in df.columns:
            result_cols.append(col(col_name))
        else:
            result_cols.append(lit(None).alias(col_name))
    return df.select(result_cols)

silver_df = safe_select(bronze_df, EXPECTED_COLUMNS)
silver_df.write.format("delta").mode("overwrite").save("/silver/orders")
```

### Schema Validation with Alerting

```python
def validate_schema(df, expected_schema: dict) -> dict:
    """Validate DataFrame schema against expected and return differences."""
    
    actual_columns = set(df.columns)
    expected_columns = set(expected_schema.keys())
    
    result = {
        "new_columns": actual_columns - expected_columns,
        "missing_columns": expected_columns - actual_columns,
        "is_valid": actual_columns == expected_columns
    }
    
    if result["new_columns"]:
        print(f"⚠️ NEW COLUMNS DETECTED: {result['new_columns']}")
        # Send alert to data team
        send_alert(f"Schema drift detected: new columns {result['new_columns']}")
    
    if result["missing_columns"]:
        print(f"🔴 MISSING COLUMNS: {result['missing_columns']}")
        raise ValueError(f"Required columns missing: {result['missing_columns']}")
    
    return result

# Usage
expected = {
    "order_id": "string",
    "customer_id": "string",
    "amount": "double",
    "order_date": "date"
}

validation = validate_schema(bronze_df, expected)
```

---

## 📊 Schema Change Types

| Change Type | Impact | Handling |
|-------------|--------|----------|
| **Add column** | Low | Auto-add at Bronze, handle NULL at Silver |
| **Remove column** | Medium | Alerting, may need ETL changes |
| **Rename column** | High | BREAKING - need code change |
| **Type change** | High | BREAKING - may lose data |

---

## 🎯 Interview Questions

| Question | Expected Answer |
|----------|----------------|
| *"New column appears - what happens?"* | Bronze accepts (mergeSchema), Silver ignores or alerts |
| *"How do you detect schema drift?"* | Compare incoming vs expected schema, alert on differences |
| *"What's backward compatibility?"* | New schema can read old data (new cols have NULL for old rows) |
| *"How does Unity Catalog help?"* | Schema registry tracks versions, enforces compatibility |

---

## 📖 Next Scenario

Continue to [Backfill Scenarios](./10-backfill-scenarios.md).
