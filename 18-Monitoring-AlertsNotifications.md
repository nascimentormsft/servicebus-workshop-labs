# Lab 18 – Monitoring: Setting Up Alerts and Notifications

## Objective

Configure **Azure Monitor alerts** to proactively detect issues in Service Bus — demonstrated with alert rules for **critical aviation messaging scenarios** where delayed detection could impact flight operations.

## Scenario

Transavia's operations must be alerted immediately when:
- Dead-letter queues start filling up (poison messages blocking processing)
- Message backlogs grow beyond acceptable thresholds (consumers failing)
- Throttling is detected (capacity limits reached)
- Queue sizes approach limits (storage running out)

Proactive alerting ensures the platform team can respond before passengers or operations are affected.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Metric alerts | **Supported** | **Supported** | **Supported** |
| Log-based alerts | **Supported** | **Supported** | **Supported** |
| Action Groups | **Supported** | **Supported** | **Supported** |

> Alert rules have their own cost (per rule per month). Consolidate alerts where possible. Premium tier's additional metrics (CPU, memory) enable more proactive alerting.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create an Action Group

Action groups define WHO gets notified and HOW:

```bash
az monitor action-group create \
  --name ag-servicebus-ops \
  --resource-group $RG \
  --short-name sb-ops \
  --action email ops-team ops-team@transavia.com
```

### Step 2 — Verify the Action Group in the Portal

1. Navigate to **Monitor** → **Alerts** → **Action groups**
2. Find `ag-servicebus-ops`
3. Click to verify the email notification is configured

### Step 3 — Create Alert: Dead-Letter Messages > 0

This alert fires when ANY dead-lettered messages appear — indicating processing failures.

```bash
SB_ID=$(az servicebus namespace show \
  --name $NAMESPACE \
  --resource-group $RG \
  --query id -o tsv)

az monitor metrics alert create \
  --name "alert-deadletter-detected" \
  --resource-group $RG \
  --scopes $SB_ID \
  --condition "total DeadletteredMessages > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "Dead-lettered messages detected in Service Bus namespace. Investigate poison messages or consumer failures." \
  --action ag-servicebus-ops
```

### Step 4 — Create Alert: Message Backlog Growing

```bash
az monitor metrics alert create \
  --name "alert-message-backlog" \
  --resource-group $RG \
  --scopes $SB_ID \
  --condition "avg ActiveMessages > 1000" \
  --window-size 15m \
  --evaluation-frequency 5m \
  --severity 2 \
  --description "Active message count exceeding 1000 for 15 minutes. Consumers may be failing or overwhelmed." \
  --action ag-servicebus-ops
```

### Step 5 — Create Alert: Throttling Detected

```bash
az monitor metrics alert create \
  --name "alert-throttling" \
  --resource-group $RG \
  --scopes $SB_ID \
  --condition "total ThrottledRequests > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 1 \
  --description "CRITICAL: Service Bus requests are being throttled. Capacity limit reached. Consider scaling to Premium or adding Messaging Units." \
  --action ag-servicebus-ops
```

> Severity 1 = Critical. Throttling directly impacts message delivery.

### Step 6 — Create Alert: Server Errors

```bash
az monitor metrics alert create \
  --name "alert-server-errors" \
  --resource-group $RG \
  --scopes $SB_ID \
  --condition "total ServerErrors > 5" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 1 \
  --description "CRITICAL: Multiple server errors detected in Service Bus. Check Azure Status page and open a support ticket if persistent." \
  --action ag-servicebus-ops
```

### Step 7 — View All Alert Rules

```bash
az monitor metrics alert list \
  --resource-group $RG \
  --output table
```

Or in the Portal:
1. Navigate to **Monitor** → **Alerts**
2. Click **Alert rules**
3. You should see all 4 rules we created

### Step 8 — Trigger a Test Alert

Trigger the dead-letter alert by sending a message that gets dead-lettered:

1. Go to a queue with `max-delivery-count` of 3 (e.g., `crew-assignments`)
2. Send a message
3. Receive → Abandon, Receive → Abandon, Receive → Abandon
4. The message is dead-lettered
5. Within 5 minutes, the `alert-deadletter-detected` rule should fire
6. Check your email for the alert notification

### Step 9 — View Fired Alerts

1. In **Monitor** → **Alerts**, you should see a fired alert
2. Click on it to see:
   - When it fired
   - The metric value that triggered it
   - The affected resource
3. Change the alert state to **Acknowledged** to indicate you're investigating

### Step 10 — Create a Log-Based Alert (Advanced)

For more complex conditions, use log-based alerts with KQL:

1. Navigate to **Monitor** → **Alerts** → **+ Create** → **Alert rule**
2. Select your Service Bus namespace as the resource
3. Condition: **Custom log search**
4. Query:

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where OperationName == "Send" and ResultType == "Error"
| summarize FailedSends = count() by bin(TimeGenerated, 5m)
| where FailedSends > 10
```

5. Alert logic: Number of results greater than 0
6. Period: 5 minutes
7. Frequency: 5 minutes
8. Assign the action group

---

## Recommended Alert Configuration for Aviation

| Alert | Metric | Threshold | Severity | Window |
|-------|--------|-----------|----------|--------|
| Dead letters detected | DeadletteredMessages | > 0 | Sev 2 (Warning) | 5 min |
| High backlog | ActiveMessages | > 1,000 | Sev 2 | 15 min |
| Critical backlog | ActiveMessages | > 10,000 | Sev 1 (Critical) | 5 min |
| Throttling | ThrottledRequests | > 0 | Sev 1 | 5 min |
| Server errors | ServerErrors | > 5 | Sev 1 | 5 min |
| Namespace near capacity | Size | > 80% of max | Sev 2 | 30 min |
| No outgoing messages | OutgoingMessages | == 0 | Sev 2 | 15 min |
| Consumer failures | AbandonedMessages | > 100 | Sev 2 | 15 min |

---

## Action Group Options

| Notification Type | Use Case |
|---|---|
| **Email** | Operations team on-call |
| **SMS** | Critical alerts for immediate attention |
| **Azure Mobile App** | Mobile push notifications |
| **Webhook** | Integration with PagerDuty, Opsgenie, Slack |
| **Azure Function** | Automated remediation (e.g., auto-scale MUs) |
| **Logic App** | Complex workflows (e.g., create incident ticket) |
| **ITSM** | ServiceNow, Cherwell integration |

---

## Cleanup — End of Workshop

If you want to clean up all workshop resources:

```bash
# WARNING: This deletes EVERYTHING in the resource group
# az group delete --name $RG --yes --no-wait
```

> Only run this after confirming with your instructor that the workshop is complete.

---

## Key Takeaways

- Proactive alerting prevents operational impact — don't wait for users to report issues
- Use metric-based alerts for real-time detection (low latency)
- Use log-based alerts for complex conditions (KQL queries)
- Combine multiple notification channels for critical alerts (email + SMS + webhook)
- Regularly review and tune alert thresholds based on actual traffic patterns

---

## Review Questions

1. What is the difference between metric-based and log-based alerts?

   > **Answer:** **Metric-based alerts** evaluate numeric metrics at regular intervals (e.g., "active messages > 1000 for 15 minutes"). They're fast (1-minute evaluation possible), low-latency, and ideal for real-time operational alerts. **Log-based alerts** run KQL queries against Log Analytics data and can evaluate complex conditions across multiple data sources (e.g., "more than 10 failed send operations from IP range X in 5 minutes"). They have higher latency (5+ minutes) but offer richer filtering and correlation capabilities.

2. Why would you use a 15-minute window for backlog alerts instead of 1 minute?

   > **Answer:** Short windows cause **alert fatigue** from false positives. A momentary spike of 1,001 messages for 30 seconds (e.g., during a batch send) would trigger a 1-minute alert even though the consumer clears it immediately. A 15-minute window ensures the backlog is **sustained**, indicating a genuine consumer problem rather than a transient burst. The trade-off: longer windows mean slower detection. For critical systems, use tiered alerts: a Warning at 15 minutes and a Critical at 5 minutes with a higher threshold.

3. How would you set up automated remediation when throttling is detected? (Hint: Azure Function + scale MUs)

   > **Answer:** (1) Create an alert rule on the `ThrottledRequests` metric. (2) Configure the action group to trigger an **Azure Function** via webhook. (3) The Function uses the Azure SDK/REST API to call `az servicebus namespace update --capacity <higher_MU>` to scale up the Premium namespace's Messaging Units. (4) Set a separate scheduled Function to scale back down after the spike (e.g., check if throttling stopped for 30 minutes, then reduce MUs). Important: include safeguards like a maximum MU cap and Slack/Teams notifications so the team is aware of automated scaling actions.
