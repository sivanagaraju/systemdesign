# Star Schema vs Snowflake Schema

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Very Common)

## The Core Question

*"Design the tables to store telemetry data for Xbox gaming sessions."*

Before writing any schema, you need to decide: **Star or Snowflake?**

---

## 🌟 Star Schema

### What Is It?

A **denormalized** design where a central **fact table** connects directly to **dimension tables**. Called "star" because the diagram looks like a star.

```
                    ┌─────────────┐
                    │ dim_player  │
                    └──────┬──────┘
                           │
    ┌─────────────┐   ┌────┴────┐   ┌─────────────┐
    │  dim_game   │───│  FACT   │───│  dim_time   │
    └─────────────┘   │ sessions │   └─────────────┘
                      └────┬────┘
                           │
                    ┌──────┴──────┐
                    │ dim_device  │
                    └─────────────┘
```

### Schema Example (Xbox Telemetry)

```sql
-- FACT TABLE: One row per gaming session
CREATE TABLE fact_gaming_sessions (
    session_id          BIGINT PRIMARY KEY,
    player_key          INT,           -- FK to dim_player
    game_key            INT,           -- FK to dim_game
    device_key          INT,           -- FK to dim_device
    date_key            INT,           -- FK to dim_date
    
    -- Measures (numeric facts)
    session_duration_sec    INT,
    achievements_unlocked   INT,
    in_game_purchases_usd   DECIMAL(10,2),
    frames_dropped          INT,
    avg_fps                 DECIMAL(5,2)
);

-- DIMENSION TABLE: Denormalized player info
CREATE TABLE dim_player (
    player_key          INT PRIMARY KEY,
    player_id           VARCHAR(50),       -- Natural key
    gamertag            VARCHAR(100),
    email               VARCHAR(255),
    country             VARCHAR(50),       -- Denormalized!
    region              VARCHAR(50),       -- Denormalized!
    subscription_tier   VARCHAR(20),
    account_created_date DATE
);
```

### Why Star Schema?

| Advantage | Explanation |
|-----------|-------------|
| **Faster Queries** | Fewer JOINs needed (dimensions are denormalized) |
| **Simple to Understand** | Business users can navigate easily |
| **Optimized for BI Tools** | Power BI, Tableau work best with star schemas |
| **Aggregate-Friendly** | Easy to create summary tables |

---

## ❄️ Snowflake Schema

### What Is It?

A **normalized** design where dimension tables are split into sub-dimensions. Called "snowflake" because the diagram branches out like a snowflake.

```
                         ┌──────────┐
                         │ country  │
                         └────┬─────┘
                              │
    ┌─────────────┐    ┌──────┴──────┐    ┌─────────────┐
    │   genre     │────│  dim_game   │    │  dim_time   │
    └─────────────┘    └──────┬──────┘    └─────────────┘
                              │
                        ┌─────┴─────┐
                        │   FACT    │
                        │  sessions │
                        └─────┬─────┘
                              │
                      ┌───────┴───────┐
                      │  dim_player   │
                      └───────┬───────┘
                              │
                      ┌───────┴───────┐
                      │ subscription  │
                      └───────────────┘
```

### Schema Example (Normalized)

```sql
-- Normalized: Player dimension split into multiple tables
CREATE TABLE dim_player (
    player_key          INT PRIMARY KEY,
    player_id           VARCHAR(50),
    gamertag            VARCHAR(100),
    email               VARCHAR(255),
    subscription_key    INT,          -- FK to dim_subscription
    country_key         INT           -- FK to dim_country
);

CREATE TABLE dim_subscription (
    subscription_key    INT PRIMARY KEY,
    tier_name           VARCHAR(20),
    monthly_price       DECIMAL(5,2),
    features            VARCHAR(500)
);

CREATE TABLE dim_country (
    country_key         INT PRIMARY KEY,
    country_name        VARCHAR(50),
    region              VARCHAR(50),
    continent           VARCHAR(30)
);
```

### Why Snowflake Schema?

| Advantage | Explanation |
|-----------|-------------|
| **Less Storage** | No duplicated dimension data |
| **Easier Updates** | Change country name in ONE place |
| **Data Integrity** | Enforced through normalization |
| **Flexible** | Easy to add new dimension attributes |

---

## ⚖️ Star vs Snowflake: The Trade-off Table

| Criteria | Star Schema | Snowflake Schema |
|----------|-------------|------------------|
| **Query Performance** | ✅ Faster (fewer JOINs) | ❌ Slower (more JOINs) |
| **Storage Space** | ❌ More (redundancy) | ✅ Less (normalized) |
| **Query Complexity** | ✅ Simple | ❌ Complex |
| **ETL Complexity** | ❌ More effort | ✅ Less effort |
| **BI Tool Compatibility** | ✅ Excellent | ⚠️ Requires modeling |
| **Dimension Updates** | ❌ Update many rows | ✅ Update one row |

---

## 🎯 Interview Answer Framework

When asked "Design tables for X", use this structure:

### 1. Clarify Requirements

> *"Before I start, I have a few questions:*
> - *Are we optimizing for query speed or storage?*
> - *What's the typical query pattern - aggregations or lookups?*
> - *How often do dimension attributes change?*
> - *Will BI tools like Power BI consume this data?"*

### 2. State Your Choice

> *"For Xbox telemetry, I'd choose a **Star Schema** because:*
> - *Gaming analytics are read-heavy (aggregations)*
> - *Query performance is critical for dashboards*
> - *Dimension updates are infrequent (player country rarely changes)*
> - *Power BI works optimally with star schemas"*

### 3. Draw the Schema

Always draw before writing SQL:

```
fact_gaming_sessions
├── FK: player_key → dim_player
├── FK: game_key → dim_game  
├── FK: device_key → dim_device
├── FK: date_key → dim_date
└── Measures: duration, achievements, purchases, fps
```

### 4. Discuss Trade-offs

> *"The trade-off is storage redundancy. If 'country' changes, I need to update all player rows. For slowly-changing attributes, I'd implement SCD Type 2."*

---

## 💡 Real-World Considerations

### When to Choose Star Schema
- OLAP workloads (analytics, reporting)
- BI tool consumption (Power BI, Tableau)
- Read-heavy query patterns
- Dimension attributes rarely change

### When to Choose Snowflake Schema
- OLTP workloads (operational systems)
- Storage cost is a major concern
- Dimension attributes change frequently
- Data warehouse with strict governance

### Hybrid Approach (Common at Microsoft)

In practice, most data warehouses use a **hybrid**:

```sql
-- Star for frequently-queried dimensions
dim_date (fully denormalized - day, week, month, quarter, year)

-- Snowflake for rarely-queried, large dimensions  
dim_product → dim_category → dim_department
```

---

## 🔥 Practice Question

**Scenario:** *"Design a schema for Microsoft Teams meeting analytics"*

**Think about:**
- What's the fact? (Meeting events? Participant joins?)
- What are the dimensions? (User, Team, Date, Meeting type)
- Would you denormalize `User.Department.Organization`?
- How would you partition this data?

---

## 📖 Next Topic

Continue to [Slowly Changing Dimensions](./02-slowly-changing-dimensions.md) to learn how to track historical changes in your dimensions.
