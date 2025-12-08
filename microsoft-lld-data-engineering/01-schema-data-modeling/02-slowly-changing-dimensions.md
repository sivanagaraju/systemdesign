# Slowly Changing Dimensions (SCD)

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Very Common at Microsoft)

## The Core Question

*"How do you track changes to customer address over time in your data warehouse?"*

This is asking about **Slowly Changing Dimensions (SCD)** - a fundamental data modeling concept.

---

## 🤔 What's the Problem?

Dimensions change over time:
- A customer **moves** to a new city
- A product **changes** category
- An employee **gets promoted**

**The question:** When the dimension changes, how do you handle historical data?

```
Customer "John" lived in Seattle from 2020-2023
Customer "John" moved to Portland in 2024

If John bought a product in 2022:
→ Should we show Seattle (his address at purchase time)?
→ Or Portland (his current address)?
```

---

## 📊 SCD Types at a Glance

| Type | Strategy | Historical Tracking | Use Case |
|------|----------|---------------------|----------|
| **Type 0** | Retain Original | None | Never changes (birth date) |
| **Type 1** | Overwrite | None | Corrections, don't need history |
| **Type 2** | Add New Row | Full history | **Most common in interviews** |
| **Type 3** | Add New Column | Limited (previous only) | Rarely used |
| **Type 4** | Mini-Dimension | Split to separate table | Rapidly changing attributes |
| **Type 6** | Hybrid (1+2+3) | Current + History | Best of both worlds |

---

## 🔵 Type 1: Overwrite

### Concept
Simply **overwrite** the old value with the new one. No history kept.

### Example

**Before:**
| player_key | gamertag | country |
|------------|----------|---------|
| 101 | xGamer42 | USA |

**After player moves:**
| player_key | gamertag | country |
|------------|----------|---------|
| 101 | xGamer42 | **Canada** |

### SQL Implementation

```sql
-- Type 1: Simple UPDATE
UPDATE dim_player
SET country = 'Canada',
    last_updated = CURRENT_TIMESTAMP
WHERE player_id = 'P12345';
```

### When to Use
- Correcting data errors
- Attributes where history doesn't matter
- Storage is extremely limited

### ⚠️ Interview Trap
> *"What if I want to analyze last month's sales by the customer's location at that time?"*

**Answer:** Type 1 won't work - you'd get the CURRENT location, not the historical one.

---

## 🟢 Type 2: Add New Row (MOST IMPORTANT)

### Concept
When a value changes, **keep the old row** and **add a new row**. Track validity with dates and flags.

### Required Columns
```sql
- surrogate_key     -- New key for each version
- natural_key       -- Original business key
- start_date        -- When this version became active
- end_date          -- When this version expired (NULL = current)
- is_current        -- Flag: 1 = active, 0 = historical
```

### Example

**Initial State:**
| player_key | player_id | gamertag | country | start_date | end_date | is_current |
|------------|-----------|----------|---------|------------|----------|------------|
| 101 | P12345 | xGamer42 | USA | 2020-01-01 | NULL | 1 |

**After player moves to Canada (2024-06-15):**
| player_key | player_id | gamertag | country | start_date | end_date | is_current |
|------------|-----------|----------|---------|------------|----------|------------|
| 101 | P12345 | xGamer42 | USA | 2020-01-01 | **2024-06-14** | **0** |
| **102** | P12345 | xGamer42 | **Canada** | **2024-06-15** | NULL | **1** |

### SQL Implementation

```sql
-- Step 1: Close out the current record
UPDATE dim_player
SET end_date = DATEADD(day, -1, '2024-06-15'),
    is_current = 0
WHERE player_id = 'P12345'
  AND is_current = 1;

-- Step 2: Insert new record
INSERT INTO dim_player (
    player_key, player_id, gamertag, country, 
    start_date, end_date, is_current
)
VALUES (
    102, 'P12345', 'xGamer42', 'Canada',
    '2024-06-15', NULL, 1
);
```

### Using MERGE (More Efficient)

```sql
MERGE INTO dim_player AS target
USING staging_players AS source
ON target.player_id = source.player_id 
   AND target.is_current = 1

-- When values changed: close old record
WHEN MATCHED AND (target.country <> source.country) THEN
    UPDATE SET 
        end_date = DATEADD(day, -1, CURRENT_DATE),
        is_current = 0

-- Insert new version (handled in separate INSERT)
;
```

### Querying Type 2 Dimensions

```sql
-- Get CURRENT dimension value
SELECT * FROM dim_player WHERE is_current = 1;

-- Get dimension value at a SPECIFIC POINT IN TIME
SELECT * 
FROM dim_player
WHERE player_id = 'P12345'
  AND '2022-03-15' BETWEEN start_date AND COALESCE(end_date, '9999-12-31');

-- Join fact table to get dimension at transaction time
SELECT 
    f.session_id,
    f.session_date,
    p.country AS country_at_session_time
FROM fact_gaming_sessions f
JOIN dim_player p 
    ON f.player_key = p.player_key
   AND f.session_date BETWEEN p.start_date AND COALESCE(p.end_date, '9999-12-31');
```

### 🎯 Key Interview Points

1. **Surrogate Key:** Each row gets a NEW player_key (101, 102, 103...)
2. **Natural Key:** The business identifier (player_id) stays the same
3. **Fact Table Join:** Facts link to the surrogate key, preserving history
4. **Performance:** Add index on `(natural_key, is_current)` for lookups

---

## 🟡 Type 3: Add New Column

### Concept
Track **only the previous value** by adding a column.

### Example

| player_key | player_id | current_country | previous_country |
|------------|-----------|-----------------|------------------|
| 101 | P12345 | Canada | USA |

### When to Use
- Only need to compare current vs previous
- Very limited history requirement
- Rarely used in practice

---

## 🟠 Type 6: Hybrid (1 + 2 + 3)

### Concept
Combines Type 1, 2, and 3 for maximum flexibility:
- **Type 2** row versioning for full history
- **Type 1** overwrite for current value in all rows
- **Type 3** previous value column

### Example

| player_key | player_id | country | current_country | start_date | end_date | is_current |
|------------|-----------|---------|-----------------|------------|----------|------------|
| 101 | P12345 | USA | **Canada** | 2020-01-01 | 2024-06-14 | 0 |
| 102 | P12345 | Canada | **Canada** | 2024-06-15 | NULL | 1 |

Notice: `current_country` is "Canada" in BOTH rows!

### Why Type 6?
- Query the current value without filtering for `is_current = 1`
- Still have full history through row versions
- Useful when most queries need current + historical together

---

## 🏗️ Design Decision Framework

When asked about SCD in an interview, use this framework:

```
┌─────────────────────────────────────────────────┐
│           Is historical tracking needed?         │
└─────────────────────┬───────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │ NO                        │ YES
        ▼                           ▼
    ┌───────┐              ┌────────────────┐
    │ Type 1│              │ How much       │
    │(Overwrite)           │ history needed?│
    └───────┘              └───────┬────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │ Previous     │ Full         │ Full + Easy
                    │ value only   │ History      │ Current Query
                    ▼              ▼              ▼
                ┌───────┐      ┌───────┐      ┌───────┐
                │ Type 3│      │ Type 2│      │ Type 6│
                └───────┘      └───────┘      └───────┘
```

---

## 💡 Azure/Databricks Specifics

### Delta Lake MERGE for SCD Type 2

```python
from delta.tables import DeltaTable

# Load existing dimension
dim_player = DeltaTable.forPath(spark, "/delta/dim_player")

# SCD Type 2 MERGE
dim_player.alias("target").merge(
    source=updates_df.alias("source"),
    condition="target.player_id = source.player_id AND target.is_current = 1"
).whenMatchedUpdate(
    condition="target.country <> source.country",
    set={
        "end_date": "source.effective_date",
        "is_current": "0"
    }
).whenNotMatchedInsert(
    values={
        "player_key": "source.player_key",
        "player_id": "source.player_id",
        "country": "source.country",
        "start_date": "source.effective_date",
        "end_date": "NULL",
        "is_current": "1"
    }
).execute()
```

---

## 🔥 Practice Question

**Scenario:** *"Xbox Game Pass subscriptions can change tiers (Basic → Standard → Ultimate). Design the dimension table and explain your SCD approach."*

**Consider:**
- Do we need to track when players were on each tier?
- What happens if we analyze "Ultimate tier gaming hours" for 2023?
- How do we handle downgrades vs upgrades?

---

## 📖 Next Topic

Continue to [Partitioning vs Bucketing](./03-partitioning-vs-bucketing.md) to learn how to optimize physical data layout.
