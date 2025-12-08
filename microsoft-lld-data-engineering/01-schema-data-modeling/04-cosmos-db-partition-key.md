# Azure Cosmos DB Partition Key Selection

> **Interview Frequency:** ⭐⭐⭐⭐⭐ (Classic Microsoft Question)

## The Core Question

*"You're storing Xbox telemetry in Cosmos DB. What partition key would you choose: DeviceID or UserID?"*

This is a **classic Microsoft interview question** testing your understanding of NoSQL partition design.

---

## 🤔 Why Does Partition Key Matter?

In Cosmos DB, the partition key determines:

1. **Data distribution** - How data is spread across physical partitions
2. **Query performance** - Cross-partition queries are expensive
3. **Throughput** - Each partition has a RU (Request Unit) limit
4. **Scalability** - Bad partition key = scalability ceiling

```
┌─────────────────────────────────────────────────────────────┐
│                    Cosmos DB Container                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │  Partition 1    │  │  Partition 2    │  │  Partition 3 │ │
│  │  DeviceID: D001 │  │  DeviceID: D002 │  │  DeviceID: D003│
│  │  ───────────────│  │  ───────────────│  │  ─────────────│ │
│  │  event1         │  │  event4         │  │  event7      │ │
│  │  event2         │  │  event5         │  │  event8      │ │
│  │  event3         │  │  event6         │  │  event9      │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚠️ The Hot Partition Problem

### What Is It?

A **hot partition** occurs when one partition receives disproportionately more traffic than others.

**Example: Bad Partition Key Choice**

```json
// Partition key: "region"
// Only 5 regions, but US has 70% of traffic!

US Partition:    ████████████████████████████ (70% of RUs, at limit!)
EU Partition:    ████████ (15% of RUs)
APAC Partition:  █████ (10% of RUs)
Other:           ██ (5% of RUs)

Result: US partition hits RU limit → requests throttled → users angry
```

### Why It Happens

| Bad Partition Key | Problem |
|-------------------|---------|
| **Low cardinality** (region, status) | Few partitions, uneven distribution |
| **Temporal** (date, hour) | Current time partition overloaded |
| **Skewed** (celebrity user_id) | Popular items overwhelm one partition |

---

## ✅ Good Partition Key Criteria

The **ideal partition key** has these properties:

| Criteria | Description | Example |
|----------|-------------|---------|
| **High Cardinality** | Many distinct values | DeviceID (millions) vs Status (3) |
| **Even Distribution** | Equal data per partition | user_id (random) vs region (skewed) |
| **Query Aligned** | Most queries include it | Avoid cross-partition queries |
| **Write Distributed** | Writes spread evenly | Avoid temporal keys for writes |

---

## 🎮 The Xbox Telemetry Decision

### Scenario Details

- 50 million Xbox devices worldwide
- Each device sends events every minute
- Users may have 1-5 devices
- 100 million registered users
- Query patterns:
  - "Get all events for a specific device"
  - "Get all events for a specific user across their devices"

### Option A: DeviceID as Partition Key

```json
{
  "id": "event-12345",
  "deviceId": "XBOX-D001-A123",  // ← Partition Key
  "userId": "user-789",
  "eventType": "game_launch",
  "timestamp": "2024-01-15T10:30:00Z",
  "gameId": "halo-infinite",
  "sessionDuration": 3600
}
```

**Pros:**
| Advantage | Explanation |
|-----------|-------------|
| ✅ High cardinality | 50 million distinct devices |
| ✅ Even distribution | Events spread across devices |
| ✅ Fast device queries | `WHERE deviceId = 'X'` hits single partition |
| ✅ Good for write scaling | Writes distributed across partitions |

**Cons:**
| Disadvantage | Explanation |
|--------------|-------------|
| ❌ Cross-partition user queries | `WHERE userId = 'Y'` scans all partitions |
| ❌ User has multiple devices | Can't get all user data in one query |

### Option B: UserID as Partition Key

```json
{
  "id": "event-12345",
  "deviceId": "XBOX-D001-A123",
  "userId": "user-789",  // ← Partition Key
  "eventType": "game_launch",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Pros:**
| Advantage | Explanation |
|-----------|-------------|
| ✅ User-centric queries | `WHERE userId = 'Y'` hits single partition |
| ✅ All devices together | User's complete activity in one partition |

**Cons:**
| Disadvantage | Explanation |
|--------------|-------------|
| ❌ Partition size skew | Power users have way more data |
| ❌ Hot partitions | Streamers/celebrities generate massive traffic |
| ❌ Device queries costly | `WHERE deviceId = 'X'` scans all partitions |

---

## 🎯 The Right Answer (Interview Setting)

> *"For Xbox telemetry, I would choose **DeviceID** as the partition key."*

### Justification:

1. **Cardinality:** 50M devices > 100M users doesn't matter; both are high enough

2. **Query Pattern Alignment:** 
   - Real-time dashboards query by device (device health monitoring)
   - Device-level queries are more common than user-level

3. **Even Distribution:**
   - Each device generates similar event volume
   - Unlike users (streamers vs casual gamers)

4. **Avoiding Hot Partitions:**
   - A popular streamer's user partition would be hot
   - But their 3 Xboxes are separate partitions

5. **Handling User Queries:**
   - Use a secondary pattern (see below)

---

## 🔧 Advanced Pattern: Composite Partition Key

### When You Need Both

Create a **composite key** when you need to query by multiple patterns:

```json
{
  "id": "event-12345",
  "partitionKey": "XBOX-D001-A123",  // DeviceID for primary pattern
  "userId": "user-789",
  "deviceId": "XBOX-D001-A123"
}
```

### Change Feed to Secondary Container

Use Cosmos DB **Change Feed** to sync data to a user-partitioned container:

```
Container 1 (Primary):     Container 2 (User View):
PartitionKey: deviceId     PartitionKey: userId
        │                         ▲
        │   ┌──────────────┐      │
        └──►│ Change Feed  │──────┘
            │  Function    │
            └──────────────┘
```

```python
# Azure Function triggered by Change Feed
@app.cosmos_db_trigger(arg_name="events", 
                       container_name="device-events",
                       database_name="telemetry")
def sync_to_user_view(events: func.DocumentList):
    for event in events:
        # Write to user-partitioned container
        user_container.upsert_item({
            "id": event["id"],
            "partitionKey": event["userId"],  # Now partitioned by user
            **event
        })
```

---

## 📊 Partition Key Sizing Guidelines

### Cosmos DB Limits

| Limit | Value |
|-------|-------|
| **Max logical partition size** | 20 GB |
| **Max RU/s per partition** | 10,000 RU/s |
| **Recommended partition count** | At least 10x expected throughput |

### Calculation Example

```
Scenario: 10 million events/day, each event = 2KB

Daily data: 10M × 2KB = 20GB/day
Yearly data: 20GB × 365 = 7.3TB

If partition key = "date":
  - 1 partition per day = 20GB (at the limit!)
  - Hot partition problem on "today"

If partition key = "deviceId" (50M devices):
  - Each device: ~200 events/day = 400KB/day
  - Even after 10 years: 1.46GB per device
  - Safely under 20GB limit ✓
```

---

## 🔥 Interview Answer Framework

### Step 1: Clarify Query Patterns

> *"What are the primary query patterns? Real-time device monitoring or user analytics?"*

### Step 2: Evaluate Options

> *"Let me evaluate both DeviceID and UserID:*
>
> *DeviceID:*
> - *50M distinct values (high cardinality ✓)*
> - *Even distribution (each device similar volume ✓)*  
> - *Aligns with device monitoring queries ✓*
>
> *UserID:*
> - *100M values (also high cardinality ✓)*
> - *Uneven distribution (power users create skew ✗)*
> - *Risk of hot partitions from celebrities ✗*"

### Step 3: Make Decision with Trade-offs

> *"I'd choose DeviceID because of even distribution. For user queries, I'd create a secondary container using Change Feed."*

### Step 4: Mention Alternatives

> *"If user queries dominate, I could use a composite key like `userId:yearMonth` to bound partition size while enabling user queries."*

---

## 💡 Common Interview Traps

### Trap 1: "But UserID has more distinct values!"

**Response:** Cardinality isn't everything. Distribution matters more. 100M users with skewed activity is worse than 50M devices with even activity.

### Trap 2: "What about date-based partition key?"

**Response:** Temporal keys create hot partitions. Current day/hour gets overwhelming traffic. Only use as a **composite** key component.

### Trap 3: "Just use id as partition key"

**Response:** Using `id` (unique per document) gives maximum distribution but makes **all queries cross-partition** unless you query by id.

---

## 🔥 Practice Question

**Scenario:** *"Design the partition key for a Cosmos DB container storing Microsoft Teams messages."*

**Consider:**
- Messages belong to conversations
- Conversations belong to teams
- Users query their recent messages
- Admin queries all team messages

What partition key would you choose? Why?

---

## 📖 Next Section

Move to [02 - Spark & Compute Internals](../02-spark-compute-internals/README.md) to learn about Spark performance optimization.
