# Azure Data Factory Deep Dive

> **Interview Frequency:** ⭐⭐⭐⭐ (Azure-Specific)

## The Core Question

*"How would you design a production data pipeline in ADF with proper error handling?"*

---

## 🏗️ ADF Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure Data Factory                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     PIPELINE                              │   │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   │   │
│  │  │ Lookup  │──►│  Copy   │──►│ Notebook│──►│ Stored  │   │   │
│  │  │Activity │   │ Activity│   │ Activity│   │  Proc   │   │   │
│  │  └─────────┘   └─────────┘   └─────────┘   └─────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                    ┌─────────┴─────────┐                        │
│                    ▼                   ▼                        │
│         ┌───────────────────┐  ┌───────────────────┐            │
│         │   Azure IR        │  │ Self-Hosted IR    │            │
│         │ (Azure resources) │  │ (On-prem/VNet)    │            │
│         └───────────────────┘  └───────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔌 Integration Runtimes

### Types of IR

| Type | Use Case | Network |
|------|----------|---------|
| **Azure IR** | Azure-to-Azure, public endpoints | Public |
| **Self-Hosted IR** | On-prem, private network | Private |
| **Azure-SSIS IR** | Run SSIS packages | Either |

### When to Use Self-Hosted IR

```
Scenario: Copy from on-premises SQL Server to ADLS

┌────────────────┐      ┌────────────────┐      ┌────────────────┐
│ On-Prem SQL    │◄────►│ Self-Hosted IR │◄────►│     ADLS      │
│ (private)      │      │ (your network) │      │ (public/pvt)   │
└────────────────┘      └────────────────┘      └────────────────┘

The IR machine must:
- Have network access to SQL Server
- Have outbound access to Azure
- Have enough CPU/memory for data movement
```

---

## 🔄 Retry Policies

### Activity-Level Retry

```json
{
    "name": "Copy_Orders",
    "type": "Copy",
    "policy": {
        "retry": 3,                    // Retry up to 3 times
        "retryIntervalInSeconds": 30,  // Wait 30s between retries
        "secureOutput": false,
        "timeout": "01:00:00"          // 1 hour max per attempt
    }
}
```

### Error Categories

| Category | Retry? | Examples |
|----------|--------|----------|
| **Transient** | Yes | Network timeout, 429 rate limit |
| **User Error** | No | Invalid credentials, wrong path |
| **System Error** | Maybe | Service unavailable |

---

## 📊 Error Handling Patterns

### Pattern 1: Try-Catch with Failure Activities

```
┌─────────────────────────────────────────────────────────────┐
│                         PIPELINE                             │
│                                                              │
│   ┌─────────┐  Success  ┌─────────┐  Success  ┌─────────┐   │
│   │  Copy   │──────────►│Transform│──────────►│  Load   │   │
│   │ Source  │           │  Data   │           │ Target  │   │
│   └────┬────┘           └────┬────┘           └─────────┘   │
│        │                     │                              │
│        │ Failure             │ Failure                      │
│        ▼                     ▼                              │
│   ┌─────────┐           ┌─────────┐                         │
│   │  Log    │           │  Log    │                         │
│   │ Error   │           │ Error   │                         │
│   └────┬────┘           └────┬────┘                         │
│        │                     │                              │
│        └──────────┬──────────┘                              │
│                   ▼                                         │
│             ┌─────────┐                                     │
│             │  Send   │                                     │
│             │  Alert  │                                     │
│             └─────────┘                                     │
└─────────────────────────────────────────────────────────────┘
```

### Pattern 2: Execute Pipeline (Child Pipeline)

```json
{
    "name": "ProcessFile",
    "type": "ExecutePipeline",
    "inputs": [],
    "pipeline": {
        "referenceName": "GenericFileProcessor",
        "type": "PipelineReference"
    },
    "parameters": {
        "sourceFile": "@item().name",
        "targetPath": "/processed/"
    },
    "waitOnCompletion": true
}
```

---

## ⚙️ Parameterized Pipelines

### Pipeline Parameters

```json
{
    "name": "DynamicCopyPipeline",
    "parameters": {
        "sourceSystem": {
            "type": "String",
            "defaultValue": "SalesDB"
        },
        "tableName": {
            "type": "String"
        },
        "watermarkColumn": {
            "type": "String",
            "defaultValue": "LastModified"
        }
    }
}
```

### Using Parameters in Activities

```json
{
    "name": "Copy_DynamicTable",
    "type": "Copy",
    "source": {
        "type": "SqlSource",
        "sqlReaderQuery": "SELECT * FROM @{pipeline().parameters.tableName} WHERE @{pipeline().parameters.watermarkColumn} > '@{activity('Lookup_Watermark').output.firstRow.lastWatermark}'"
    },
    "sink": {
        "type": "ParquetSink",
        "storeSettings": {
            "type": "AzureBlobFSWriteSettings"
        },
        "formatSettings": {
            "type": "ParquetWriteSettings"
        }
    }
}
```

---

## 🔁 Incremental Load Pattern

```
┌─────────────────────────────────────────────────────────────┐
│              INCREMENTAL COPY PIPELINE                       │
│                                                              │
│  1. ┌──────────────┐                                        │
│     │   Lookup     │  ← Get last watermark from control     │
│     │  Watermark   │    table                               │
│     └──────┬───────┘                                        │
│            │                                                │
│            ▼                                                │
│  2. ┌──────────────┐                                        │
│     │  Copy Data   │  ← WHERE ModifiedDate > @lastWatermark │
│     │ (Incremental)│                                        │
│     └──────┬───────┘                                        │
│            │                                                │
│            ▼                                                │
│  3. ┌──────────────┐                                        │
│     │   Stored     │  ← Update control table with new       │
│     │   Procedure  │    watermark = MAX(ModifiedDate)       │
│     └──────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
```

### Control Table

```sql
CREATE TABLE WatermarkControl (
    TableName VARCHAR(100) PRIMARY KEY,
    LastWatermark DATETIME,
    LastRunTime DATETIME,
    RowsProcessed INT
);
```

### Lookup Activity

```json
{
    "name": "Lookup_Watermark",
    "type": "Lookup",
    "source": {
        "type": "SqlSource",
        "sqlReaderQuery": "SELECT LastWatermark FROM WatermarkControl WHERE TableName = '@{pipeline().parameters.tableName}'"
    }
}
```

---

## 📧 Alerting Configuration

### Web Activity to Send Alert

```json
{
    "name": "SendAlertOnFailure",
    "type": "WebActivity",
    "method": "POST",
    "url": "https://prod-xx.westus.logic.azure.com:443/workflows/...",
    "body": {
        "pipelineName": "@{pipeline().Pipeline}",
        "runId": "@{pipeline().RunId}",
        "status": "Failed",
        "errorMessage": "@{activity('Copy_Data').error.message}",
        "triggeredTime": "@{pipeline().TriggerTime}"
    }
}
```

---

## 🎯 Interview Answer Framework

When asked about ADF design:

> **Integration Runtime selection:**
> *"Use Azure IR for Azure-to-Azure. Use Self-Hosted IR for on-premises or private network access. Size the IR based on data volume and parallel copy operations."*

> **Error handling:**
> *"Implement retry policies at activity level (3 retries with 30s intervals). Use upon-failure paths for logging and alerting. Don't let transient errors fail the pipeline."*

> **Parameterization:**
> *"Make pipelines reusable with parameters for source/target, table names, and watermark columns. One pipeline can process multiple tables."*

> **Incremental loading:**
> *"Use control table to track watermarks. Lookup last watermark → Copy new/changed records → Update watermark. This pattern handles late-arriving data."*

---

## 📖 Next Section

Move to [04 - OOP Design Patterns](../04-oop-design-patterns/README.md) for software engineering LLD topics.
