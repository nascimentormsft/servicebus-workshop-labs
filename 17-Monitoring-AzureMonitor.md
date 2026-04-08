# Lab 17 – Monitoring: Azure Monitor for Service Bus

## Objective

Set up **Azure Monitor** to track Service Bus health, performance, and usage metrics — demonstrated with a **real-time operations dashboard** for Transavia's messaging infrastructure.

## Scenario

Transavia's platform team needs to monitor the health of all Service Bus queues and topics across all environments. They need visibility into message throughput, queue depths, error rates, and resource utilization to proactively detect issues before they impact flight operations.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Azure Monitor metrics | **Supported** | **Supported** | **Supported** |
| Diagnostic logs | **Supported** | **Supported** | **Supported** |
| Log Analytics integration | **Supported** | **Supported** | **Supported** |
| CPU/Memory metrics | N/A | N/A | **Premium only** |

> Premium tier exposes **CPU** and **Memory** metrics that Standard doesn't — essential for capacity planning and proactive scaling.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Explore Built-in Metrics

1. Navigate to your Service Bus namespace in the Portal
2. Go to **Monitoring** → **Metrics**
3. Click **Add metric** and explore the available metrics:

| Metric | Description | Use Case |
|--------|-------------|----------|
| **Active Messages** | Messages in queue/subscription ready for delivery | Queue depth monitoring |
| **Dead-lettered Messages** | Messages in DLQ | Error detection |
| **Incoming Messages** | Messages sent to the entity | Throughput monitoring |
| **Outgoing Messages** | Messages delivered to consumers | Consumer health |
| **Completed Messages** | Messages successfully completed | Processing verification |
| **Abandoned Messages** | Messages abandoned by consumers | Consumer issue detection |
| **Throttled Requests** | Requests rejected due to throttling | Capacity issues |
| **Server Errors** | Server-side errors | Platform issues |
| **Size** | Current size of entity | Capacity planning |

### Step 2 — Create a Multi-Metric Chart

1. In the Metrics blade, add the first metric: **Incoming Messages** (Sum)
2. Click **Add metric** → **Outgoing Messages** (Sum)
3. Click **Add metric** → **Dead-lettered Messages** (Sum)
4. Set the time range to **Last 4 hours**
5. Set granularity to **5 minutes**
6. You now have a single chart showing message flow and dead-letter accumulation

### Step 3 — Filter Metrics by Entity

1. Click **Add filter**
2. Property: **Entity Name**
3. Operator: `=`
4. Values: Select `booking-confirmations`
5. Now the chart shows metrics only for that specific queue

### Step 4 — Split Metrics by Entity

1. Remove the filter from Step 3
2. Click **Apply splitting**
3. Split by: **Entity Name**
4. Now you see each queue/topic on a separate line — great for comparing load across entities

### Step 5 — Enable Diagnostic Logs

```bash
# Create a Log Analytics workspace (if you don't have one)
az monitor log-analytics workspace create \
  --workspace-name law-servicebus-workshop \
  --resource-group $RG \
  --location westeurope

# Get the workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --workspace-name law-servicebus-workshop \
  --resource-group $RG \
  --query id -o tsv)

# Get Service Bus namespace resource ID
SB_ID=$(az servicebus namespace show \
  --name $NAMESPACE \
  --resource-group $RG \
  --query id -o tsv)

# Enable diagnostic settings
az monitor diagnostic-settings create \
  --name "sb-diagnostics" \
  --resource $SB_ID \
  --workspace $WORKSPACE_ID \
  --logs '[{"category": "OperationalLogs", "enabled": true}, {"category": "VNetAndIPFilteringLogs", "enabled": true}, {"category": "RuntimeAuditLogs", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

### Step 6 — Verify Diagnostic Settings in Portal

1. Navigate to your Service Bus namespace
2. Go to **Monitoring** → **Diagnostic settings**
3. You should see `sb-diagnostics` enabled
4. Click on it to review which log categories and metrics are being captured

### Step 7 — Query Logs with KQL (Kusto Query Language)

1. Navigate to your Log Analytics workspace
2. Click **Logs**
3. Run these queries:

**All operational events in the last hour:**
```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where TimeGenerated > ago(1h)
| project TimeGenerated, OperationName, Resource, ResultType, _ResourceId
| order by TimeGenerated desc
```

**Failed operations:**
```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where ResultType != "Success" and ResultType != ""
| summarize count() by OperationName, ResultType
| order by count_ desc
```

**Message count by entity:**
```kql
AzureMetrics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where MetricName == "ActiveMessages"
| summarize MaxMessages = max(Maximum) by EntityName = tostring(split(_ResourceId, "/")[-1])
| order by MaxMessages desc
```

### Step 8 — Pin to Azure Dashboard

1. After creating a useful chart in Metrics, click **Pin to dashboard**
2. Select or create a dashboard: "Transavia Service Bus Operations"
3. Repeat for different metrics to build a comprehensive operations view

### Step 9 — Monitor Namespace Health Overview

1. Navigate to your namespace → **Overview**
2. The overview blade shows:
   - Request count graph
   - Message count graph
   - Size usage graph
3. These give a quick health check without going into detailed metrics

### Step 10 — Set Up Workbooks for Deep Analysis

1. Navigate to **Monitoring** → **Workbooks**
2. Click **+ New**
3. Add a query step with:

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where TimeGenerated > ago(24h)
| summarize
    TotalOperations = count(),
    FailedOperations = countif(ResultType != "Success"),
    SuccessRate = round(100.0 * countif(ResultType == "Success") / count(), 2)
    by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

4. Set visualization to **Line chart**
5. Save the workbook as "Service Bus Health Report"

---

## Key Metrics for Aviation Operations

| Metric | Threshold | Action |
|--------|-----------|--------|
| Active Messages > 10,000 | Warning | Check consumer health |
| Dead-lettered Messages > 0 | Warning | Investigate poison messages |
| Throttled Requests > 0 | Critical | Scale up or optimize |
| Server Errors > 0 | Critical | Check Azure status / open support ticket |
| Size > 80% capacity | Warning | Archive old messages or increase size |

---

## Key Takeaways

- Azure Monitor provides metrics (real-time) and diagnostic logs (detailed audit)
- Use Metrics for dashboards and quick health checks
- Use Log Analytics and KQL for deep investigation
- Enable diagnostic logs for compliance and security auditing
- Premium tier provides additional CPU/Memory metrics

---

## Review Questions

1. What is the difference between Azure Monitor Metrics and Diagnostic Logs?

   > **Answer:** **Metrics** are lightweight, near-real-time numeric values (e.g., message count, request count) sampled at regular intervals. They're stored for 93 days and are ideal for dashboards, alerts, and quick health checks. **Diagnostic Logs** are detailed, event-level records (e.g., "User X sent a message to queue Y at time Z with result Success"). They capture the full audit trail and are sent to Log Analytics, Storage, or Event Hubs. Use metrics for monitoring; use logs for investigation and compliance.

2. How long are metrics retained by default? How about diagnostic logs?

   > **Answer:** **Metrics** are retained for **93 days** automatically at no extra cost. **Diagnostic logs** retention depends on the destination: Log Analytics workspace retention is configurable (default 30 days, max 730 days); Azure Storage retains them indefinitely until you delete them; Event Hubs streams them for real-time processing. For aviation compliance, configure Log Analytics retention to at least 90 days and archive to Storage for long-term (years).

3. Why are CPU and Memory metrics only available on Premium tier?

   > **Answer:** Standard tier runs on **shared infrastructure** — your namespace doesn't have dedicated CPU or memory allocations, so there's nothing meaningful to measure per-namespace. Premium tier provides **dedicated Messaging Units** with their own compute and memory resources. CPU and Memory metrics on Premium reflect your dedicated resource utilization and are essential for capacity planning: if CPU consistently exceeds 70%, you should scale up MUs before performance degrades.
