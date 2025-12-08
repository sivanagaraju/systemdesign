# 13 - Advanced Data Modeling

> **Specialized modeling patterns**

---

## ⏰ Time-Series Modeling

**Use case:** IoT telemetry, metrics, sensor data

```sql
CREATE TABLE device_telemetry (
    device_id STRING,
    event_time TIMESTAMP,
    temperature DOUBLE,
    humidity DOUBLE
)
PARTITIONED BY (event_date DATE)
CLUSTERED BY (device_id) INTO 256 BUCKETS;
```

**Best Practices:**
- Partition by time (day/hour)
- Bucket by device for aggregations
- Pre-aggregate for dashboards (rollups)
- Consider time-series DBs for hot data

---

## 🕸️ Graph Data Patterns

**Use case:** Social networks, org charts, fraud detection

**When to use graph:**
- Many-to-many relationships
- Path/traversal queries ("friends of friends")
- Network analysis

**Cosmos DB Gremlin:**
```groovy
// Find friends of friends
g.V().has('name','John')
 .out('knows')
 .out('knows')
 .values('name')
```

---

## 🏢 Multi-Tenancy Design

| Strategy | Isolation | Cost | Complexity |
|----------|-----------|------|------------|
| **Shared tables** | Low | Low | Filter by tenant_id |
| **Schema per tenant** | Medium | Medium | Separate schemas |
| **DB per tenant** | High | High | Complete isolation |

**Row-Level Security:**
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
FOR SELECT USING (tenant_id = current_tenant());
```
