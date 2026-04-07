# Lab 05 – Filtering Messages with Subscription Rules

## Objective

Create **subscription filters** on a Service Bus Topic to route **flight operation messages** to the right systems based on message properties — so each subscriber only receives relevant messages.

## Scenario

Transavia's operations center publishes all operational events to a single `flight-operations` topic. Different systems need different subsets:
- The **delay management** team only cares about delays and cancellations
- The **gate management** system only needs gate-change events at AMS (Schiphol)
- The **crew system** needs all events for specific aircraft types

Subscription filters let each subscriber define which messages they receive, reducing noise and processing overhead.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| SQL Filters | N/A | **Supported** | **Supported** |
| Correlation Filters | N/A | **Supported** | **Supported** |
| Max Rules per Subscription | N/A | 2,000 | 2,000 |

> **Correlation Filters** are more performant than SQL Filters because they use hash-based matching. Use Correlation Filters when you only need exact-match on properties. Use SQL Filters for complex expressions (ranges, LIKE, AND/OR).

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create the Flight Operations Topic

```bash
az servicebus topic create \
  --name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --max-size 1024
```

### Step 2 — Create Subscriptions

```bash
# Delay management - wants only delays and cancellations
az servicebus topic subscription create \
  --name delay-management \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG

# Gate management - wants gate changes at AMS only
az servicebus topic subscription create \
  --name gate-management-ams \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG

# Crew system - wants all events (no filter)
az servicebus topic subscription create \
  --name crew-system \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG
```

### Step 3 — Remove the Default "Accept All" Rule

Each subscription is created with a default `$Default` rule that accepts all messages. Remove it before adding filters:

```bash
# Remove default rule from delay-management
az servicebus topic subscription rule delete \
  --name '$Default' \
  --subscription-name delay-management \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG

# Remove default rule from gate-management-ams
az servicebus topic subscription rule delete \
  --name '$Default' \
  --subscription-name gate-management-ams \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG
```

> **Leave** the `$Default` rule on `crew-system` — it should receive all messages.

### Step 4 — Add a SQL Filter for Delay Management

```bash
az servicebus topic subscription rule create \
  --name delay-and-cancellation-filter \
  --subscription-name delay-management \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --filter-sql-expression "eventType = 'Delay' OR eventType = 'Cancellation'"
```

### Step 5 — Add a Correlation Filter for Gate Management

```bash
az servicebus topic subscription rule create \
  --name gate-change-ams-filter \
  --subscription-name gate-management-ams \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --filter-sql-expression "eventType = 'GateChange' AND airport = 'AMS'"
```

### Step 6 — Verify Rules in the Portal

1. Navigate to **Topics** → `flight-operations`
2. Click on the `delay-management` subscription
3. Go to **Rules** — you should see `delay-and-cancellation-filter`
4. Repeat for `gate-management-ams`

### Step 7 — Send Test Messages

Using Service Bus Explorer on the `flight-operations` topic, send these messages with **Custom Properties**:

**Message 1 — Delay event at AMS:**
- Custom Properties: `eventType` = `Delay`, `airport` = `AMS`
```json
{
  "flightNumber": "HV6321",
  "event": "Delay",
  "airport": "AMS",
  "originalTime": "14:30",
  "newTime": "16:00",
  "reason": "Weather"
}
```

**Message 2 — Gate change at AMS:**
- Custom Properties: `eventType` = `GateChange`, `airport` = `AMS`
```json
{
  "flightNumber": "HV5102",
  "event": "GateChange",
  "airport": "AMS",
  "oldGate": "D47",
  "newGate": "E12"
}
```

**Message 3 — Gate change at BCN (not AMS):**
- Custom Properties: `eventType` = `GateChange`, `airport` = `BCN`
```json
{
  "flightNumber": "HV6321",
  "event": "GateChange",
  "airport": "BCN",
  "oldGate": "B22",
  "newGate": "B25"
}
```

**Message 4 — Cancellation:**
- Custom Properties: `eventType` = `Cancellation`, `airport` = `AMS`
```json
{
  "flightNumber": "HV1234",
  "event": "Cancellation",
  "airport": "AMS",
  "reason": "Technical issue"
}
```

### Step 8 — Verify Filter Results

Check the message count for each subscription:

| Subscription | Expected Messages | Explanation |
|---|---|---|
| `delay-management` | 2 | Msg 1 (Delay) + Msg 4 (Cancellation) |
| `gate-management-ams` | 1 | Msg 2 (GateChange at AMS only) |
| `crew-system` | 4 | All messages (no filter) |

```bash
# Verify counts
az servicebus topic subscription show \
  --name delay-management \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"

az servicebus topic subscription show \
  --name gate-management-ams \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"

az servicebus topic subscription show \
  --name crew-system \
  --topic-name flight-operations \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"
```

---

## Filter Types Summary

| Filter Type | Use Case | Performance |
|-------------|----------|-------------|
| **SQL Filter** | Complex expressions (`AND`, `OR`, `LIKE`, `IN`, comparisons) | Slower (expression evaluation) |
| **Correlation Filter** | Exact match on one or more properties | Faster (hash-based lookup) |
| **True Filter** (`1=1`) | Accept all messages (same as default rule) | N/A |
| **False Filter** (`1=0`) | Reject all messages (useful for pausing) | N/A |

---

## Review Questions

1. What happens to a message that doesn't match any subscription's filter?
2. Why should you remove the `$Default` rule before adding custom filters?
3. When would you choose a Correlation Filter over a SQL Filter?
