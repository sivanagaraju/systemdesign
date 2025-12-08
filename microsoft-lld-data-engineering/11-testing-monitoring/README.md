# 11 - Testing & Monitoring

> **Quality engineering for data pipelines**

---

## 🧪 Testing Pyramid

```
        ┌───────────────┐
        │  Integration  │   Few, expensive
        │    Tests      │
        ├───────────────┤
        │  Unit Tests   │   Many, cheap
        │  (Transform)  │
        └───────────────┘
```

---

## ✅ Unit Testing Transformations

```python
import pytest
from pyspark.sql import SparkSession

def test_amount_calculation(spark):
    # Arrange
    input_data = [{"quantity": 10, "price": 5.0}]
    input_df = spark.createDataFrame(input_data)
    
    # Act
    result_df = calculate_amount(input_df)
    
    # Assert
    assert result_df.collect()[0]["amount"] == 50.0
```

---

## 📊 Data Observability

| Metric | What to Monitor |
|--------|-----------------|
| **Freshness** | When was data last updated? |
| **Volume** | Row count within expected range? |
| **Schema** | Did columns change? |
| **Distribution** | Did null ratio spike? |

---

## 🔔 Alerting Best Practices

- **Severity levels:** P1 (page), P2 (escalate), P3 (ticket)
- **Avoid fatigue:** Group related alerts
- **Runbooks:** Link to resolution steps
- **Business hours:** Route appropriately
