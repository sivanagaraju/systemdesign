# 03 - Microsoft & Azure CDC Ecosystem

> **"Know the tools in your employer's toolbox."**

For a Microsoft interview, generic knowledge isn't enough. You must demonstrate deep familiarity with the Azure data estate.

---

## 1. SQL Server CDC (The Standard Standard)

SQL Server has a robust, built-in CDC engine that has been the industry gold standard for years.

### How it Works (Internals)
1.  **Transaction Log**: Every insert/update/delete writes to the T-Log.
2.  **Capture Job**: A background process (SQL Agent Job) scans the T-Log for changes to "CDC-enabled" tables.
3.  **Change Tables**: It writes these changes to special system tables (e.g., `cdc.dbo_Users_CT`).
4.  **Functions**: You don't query the tables directly; you use table-valued functions like `cdc.fn_cdc_get_net_changes_...`.

### ⚠️ Staff-Level Considerations
*   **The "Log Truncation" Risk**: If the Capture Job stops, the T-Log **cannot truncate**. It will grow until it consumes all disk space, crashing the database.
*   **Performance Overhead**: Enabling CDC adds roughly 10-15% CPU/IO overhead because every write is written twice (once to T-Log, once to Change Table).
*   **Cleanup**: There is a separate "Cleanup Job" that deletes data from Change Tables (default retention: 3 days). If this fails, your database grows recursively.

---

## 2. Azure Cosmos DB Change Feed (The "Cloud Native" Stream)

This is one of the most powerful features in Azure. Unlike SQL Server's "bolt-on" CDC, Cosmos DB was designed with the Change Feed as a core primitive.

### Key Concepts
*   **Persistent Log**: The Change Feed is not a side-effect; it is the log itself. It is available by default (no performance penalty to "enable" it).
*   **Ordered by Partition**: Events are guaranteed to be ordered **within a Partition Key**.
*   **Events**: Currently, it captures the *latest version* of the document.
    *   *Note*: "Full Fidelity" change feed (capturing intermediate updates and deletes) is in Preview (as of late 2024/2025).

### Architecture Pattern: Event Sourcing & Lambda
You can use Azure Functions (with a Cosmos DB Trigger) to listen to the Change Feed.
*   **Microservices**: One service updates a document; another service triggers off the change to send an email.
*   **Materialized Views**: Use the Change Feed to replicate data from a "Write-Optimized" container to a "Read-Optimized" SQL database or Search Index.

---

## 3. Azure Data Factory (ADF) & CDC

ADF is the orchestration engine. It has native CDC capabilities ("CDC Resources").

### The "Top-Level" CDC Resource
*   **Abstracted Complexity**: You don't write offsets or manage watermarks manually.
*   **Sources**: Supports SQL Server, Oracle, Cosmos DB.
*   **Latency**: It allows you to define "Latency" vs. "Throughput" trade-offs.

### Interview Pivot
If asked: *"How do we get data from On-Prem SQL to Delta Lake?"*
*   **Answer**: "I would create a **Self-Hosted Integration Runtime** near the SQL Server. I'd enable SQL Server CDC on the required tables. Then, I'd use an **ADF Pipeline** with a Copy Activity using the `sys.fn_cdc_get_net_changes` logic or the native **ADF CDC resource** to micro-batch changes into ADLS Gen2 Delta tables."

---

## 4. Debezium on Azure (The "Open Source" Option)

Sometimes native tools aren't enough (e.g., need Kafka, need Postgres/MySQL).

*   **Architecture**:
    *   Run Debezium implementations as **Kafka Connect** clusters on **Azure Kubernetes Service (AKS)**.
    *   Use **Azure Event Hubs** (Kafka compatible) as the message broker.
    *   **Pros**: Vendor agnostic, extremely high throughput, highly tunable.
    *   **Cons**: High operational burden (managing Kubernetes + Kafka Connect clusters).
