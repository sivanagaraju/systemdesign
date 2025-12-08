# Data Reconciliation

> **Validating your pipeline didn't lose data**

## The Core Problem

*"Source says 1 million records, but you only have 999,500. How do you find the missing 500?"*

```
Source System:                     Your Data Lake:
───────────────────────            ───────────────────────
Total records: 1,000,000           Total records: 999,500
Total amount: $15,234,567.89       Total amount: $15,229,123.45

MISMATCH! Where are the missing 500 records and $5,444.44?
```

---

## 🏗️ Reconciliation Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATA RECONCILIATION ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  RECONCILIATION POINTS                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                                                                          ││
│  │  Source ───► Bronze ───► Silver ───► Gold ───► Serving                  ││
│  │     │           │           │          │           │                     ││
│  │     │           │           │          │           │                     ││
│  │     ▼           ▼           ▼          ▼           ▼                     ││
│  │  ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐                    ││
│  │  │Count│    │Count│    │Count│    │Count│    │Count│                    ││
│  │  │ Sum │    │ Sum │    │ Sum │    │ Sum │    │ Sum │                    ││
│  │  │Check│    │Check│    │Check│    │Check│    │Check│                    ││
│  │  └─────┘    └─────┘    └─────┘    └─────┘    └─────┘                    ││
│  │                                                                          ││
│  │  Compare adjacent pairs to find WHERE data was lost                      ││
│  │                                                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  RECONCILIATION REPORT TABLE                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                                                                          ││
│  │  ┌────────────┬────────────┬────────────┬────────────┬─────────────────┐││
│  │  │ check_date │ layer      │ row_count  │ sum_amount │ status          │││
│  │  ├────────────┼────────────┼────────────┼────────────┼─────────────────┤││
│  │  │ 2024-01-15 │ source     │ 1,000,000  │ 15,234,567 │ -               │││
│  │  │ 2024-01-15 │ bronze     │ 1,000,000  │ 15,234,567 │ ✅ MATCH        │││
│  │  │ 2024-01-15 │ silver     │ 999,500    │ 15,229,123 │ ❌ MISMATCH     │││
│  │  │ 2024-01-15 │ gold       │ 999,500    │ 15,229,123 │ ✅ MATCH        │││
│  │  └────────────┴────────────┴────────────┴────────────┴─────────────────┘││
│  │                                                                          ││
│  │  Silver has mismatch → Check Bronze-to-Silver transformation!           ││
│  │                                                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Code Implementation

### Reconciliation Framework

```python
from pyspark.sql.functions import *
from dataclasses import dataclass
from typing import Optional

@dataclass
class ReconciliationResult:
    layer: str
    check_date: str
    row_count: int
    sum_amount: float
    distinct_keys: int
    checksum: str  # Hash of all primary keys

class DataReconciler:
    """
    Compare row counts and checksums between layers.
    """
    
    def __init__(self, spark):
        self.spark = spark
    
    def collect_metrics(self, layer: str, table_path: str, 
                        date_col: str, check_date: str,
                        key_col: str, amount_col: str) -> ReconciliationResult:
        """Collect reconciliation metrics for a single layer."""
        
        df = self.spark.read.format("delta").load(table_path) \
            .filter(col(date_col) == check_date)
        
        # Collect metrics
        metrics = df.agg(
            count("*").alias("row_count"),
            sum(amount_col).alias("sum_amount"),
            countDistinct(key_col).alias("distinct_keys"),
            # Checksum: sorted hash of all keys
            md5(concat_ws(",", sort_array(collect_list(key_col)))).alias("checksum")
        ).collect()[0]
        
        return ReconciliationResult(
            layer=layer,
            check_date=check_date,
            row_count=metrics["row_count"],
            sum_amount=float(metrics["sum_amount"] or 0),
            distinct_keys=metrics["distinct_keys"],
            checksum=metrics["checksum"]
        )
    
    def compare_layers(self, result1: ReconciliationResult, 
                       result2: ReconciliationResult) -> dict:
        """Compare two layers and identify discrepancies."""
        
        return {
            "layer1": result1.layer,
            "layer2": result2.layer,
            "row_count_diff": result2.row_count - result1.row_count,
            "sum_amount_diff": result2.sum_amount - result1.sum_amount,
            "distinct_keys_diff": result2.distinct_keys - result1.distinct_keys,
            "checksum_match": result1.checksum == result2.checksum,
            "is_match": (
                result1.row_count == result2.row_count and
                abs(result1.sum_amount - result2.sum_amount) < 0.01
            )
        }
    
    def find_missing_records(self, source_path: str, target_path: str,
                             key_col: str, date_col: str, 
                             check_date: str) -> "DataFrame":
        """Find specific records that exist in source but not target."""
        
        source_df = self.spark.read.format("delta").load(source_path) \
            .filter(col(date_col) == check_date) \
            .select(key_col).distinct()
        
        target_df = self.spark.read.format("delta").load(target_path) \
            .filter(col(date_col) == check_date) \
            .select(key_col).distinct()
        
        # Records in source but not in target
        missing = source_df.subtract(target_df)
        
        return missing
    
    def run_full_reconciliation(self, check_date: str):
        """Run reconciliation across all layers."""
        
        layers = [
            ("bronze", "/bronze/orders", "ingestion_date"),
            ("silver", "/silver/orders", "order_date"),
            ("gold", "/gold/order_aggregates", "order_date"),
        ]
        
        results = []
        for layer, path, date_col in layers:
            result = self.collect_metrics(
                layer=layer,
                table_path=path,
                date_col=date_col,
                check_date=check_date,
                key_col="order_id",
                amount_col="amount"
            )
            results.append(result)
            print(f"{layer}: {result.row_count} rows, ${result.sum_amount:,.2f}")
        
        # Compare adjacent layers
        for i in range(len(results) - 1):
            comparison = self.compare_layers(results[i], results[i+1])
            if not comparison["is_match"]:
                print(f"❌ MISMATCH between {comparison['layer1']} and {comparison['layer2']}")
                print(f"   Row diff: {comparison['row_count_diff']}")
                print(f"   Amount diff: ${comparison['sum_amount_diff']:,.2f}")
            else:
                print(f"✅ {comparison['layer1']} → {comparison['layer2']} matches")


# Usage
reconciler = DataReconciler(spark)
reconciler.run_full_reconciliation("2024-01-15")

# Find specific missing records
missing = reconciler.find_missing_records(
    source_path="/bronze/orders",
    target_path="/silver/orders", 
    key_col="order_id",
    date_col="order_date",
    check_date="2024-01-15"
)
print(f"Missing {missing.count()} records:")
missing.show()
```

### Automated Daily Reconciliation

```python
# Daily job: Write reconciliation results to tracking table

def daily_reconciliation(check_date: str):
    reconciler = DataReconciler(spark)
    
    layers = [
        ("source_api", get_source_count, get_source_sum),  # API call
        ("bronze", "/bronze/orders", "ingestion_date"),
        ("silver", "/silver/orders", "order_date"),
    ]
    
    results = []
    for layer_config in layers:
        metrics = reconciler.collect_metrics(...)
        results.append(metrics)
    
    # Write to tracking table
    results_df = spark.createDataFrame([
        (r.layer, r.check_date, r.row_count, r.sum_amount, r.checksum)
        for r in results
    ], ["layer", "check_date", "row_count", "sum_amount", "checksum"])
    
    results_df.write.format("delta") \
        .mode("append") \
        .save("/monitoring/reconciliation_results")
    
    # Alert on mismatch
    if not all_match(results):
        send_alert("Data reconciliation failed for " + check_date)
```

---

## 📊 Reconciliation Checks

| Check Type | What It Validates | When to Use |
|------------|-------------------|-------------|
| **Row count** | No records lost | Always |
| **Sum of amounts** | Financial accuracy | Financial data |
| **Distinct keys** | No missing unique records | Primary key data |
| **Checksum** | Exact record match | High accuracy required |

---

## 🎯 Interview Questions

| Question | Expected Answer |
|----------|----------------|
| *"How do you validate no data loss?"* | Compare row counts between layers, checksum on key columns |
| *"Where do you run reconciliation?"* | Between each layer transition (Bronze→Silver→Gold) |
| *"What if counts don't match?"* | Find missing records with SUBTRACT, check DLQ, debug transformation |
| *"How often to reconcile?"* | Daily for critical data, match with source before Sign-off |

---

## 📖 Next Scenario

Continue to [Hot Partitions](./13-hot-partitions.md).
