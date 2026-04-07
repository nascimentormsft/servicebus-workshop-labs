# Lab 13 – Auto-Forwarding

## Objective

Configure **auto-forwarding** to chain queues and subscriptions, creating message pipelines — demonstrated with a **multi-stage flight disruption handling** system where disruption events flow through triage, processing, and notification stages.

## Scenario

When a flight is disrupted (delay > 2 hours, cancellation, diversion), the event flows through stages:
1. **Triage queue** — classifies the disruption severity
2. **Processing queue** — determines re-accommodation options (rebooking, hotel, compensation)
3. **Notification queue** — sends passenger communications

Auto-forwarding connects these stages without consumer code needing to explicitly send to the next queue.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Auto-Forwarding | Not supported | **Supported** | **Supported** |
| Cross-namespace forwarding | N/A | Not supported | Not supported |

> Auto-forwarding only works within the same namespace. For cross-namespace routing, use Azure Functions or Logic Apps.

> **Important:** Auto-forwarding counts as a send operation — it does NOT count against namespace outgoing message limits.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create the Pipeline Queues

```bash
# Stage 3: Notification (create first — it's the destination)
az servicebus queue create \
  --name disruption-notification \
  --namespace-name $NAMESPACE \
  --resource-group $RG

# Stage 2: Processing (forwards to notification)
az servicebus queue create \
  --name disruption-processing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --forward-to disruption-notification

# Stage 1: Triage (forwards to processing)
az servicebus queue create \
  --name disruption-triage \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --forward-to disruption-processing
```

> **Order matters:** Create the destination queue first. The `--forward-to` target must already exist.

### Step 2 — Verify Auto-Forwarding in the Portal

1. Navigate to **Queues** → `disruption-triage`
2. In **Properties**, find **Forward messages to**: `disruption-processing`
3. Check `disruption-processing` → **Forward messages to**: `disruption-notification`

### Step 3 — Send a Disruption Event to Triage

Using Service Bus Explorer on `disruption-triage`:

```json
{
  "disruptionId": "DISR-20260407-001",
  "flightNumber": "HV6321",
  "route": "AMS-BCN",
  "type": "Delay",
  "delayMinutes": 180,
  "reason": "Technical issue",
  "passengersAffected": 189,
  "timestamp": "2026-04-07T12:00:00Z"
}
```

### Step 4 — Trace the Message Through the Pipeline

After sending, check where the message ended up:

```bash
# Stage 1: Triage — should be 0 (auto-forwarded)
az servicebus queue show \
  --name disruption-triage \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"

# Stage 2: Processing — should be 0 (auto-forwarded)
az servicebus queue show \
  --name disruption-processing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"

# Stage 3: Notification — should be 1 (final destination)
az servicebus queue show \
  --name disruption-notification \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"
```

> The message automatically traversed the entire chain and landed in `disruption-notification`.

### Step 5 — Auto-Forward from Topic Subscription to Queue

A common pattern: fan-out from a topic, then forward specific subscriptions to dedicated queues.

```bash
# Create a priority disruption queue
az servicebus queue create \
  --name disruption-priority \
  --namespace-name $NAMESPACE \
  --resource-group $RG

# Create a topic for all disruptions
az servicebus topic create \
  --name all-disruptions \
  --namespace-name $NAMESPACE \
  --resource-group $RG

# Create a subscription that auto-forwards severe disruptions to the priority queue
az servicebus topic subscription create \
  --name severe-disruptions \
  --topic-name all-disruptions \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --forward-to disruption-priority

# Add a filter: only cancellations and long delays
az servicebus topic subscription rule delete \
  --name '$Default' \
  --subscription-name severe-disruptions \
  --topic-name all-disruptions \
  --namespace-name $NAMESPACE \
  --resource-group $RG

az servicebus topic subscription rule create \
  --name severe-filter \
  --subscription-name severe-disruptions \
  --topic-name all-disruptions \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --filter-sql-expression "type = 'Cancellation' OR delayMinutes > 120"
```

### Step 6 — Test the Topic-to-Queue Forwarding

Send a message to `all-disruptions` topic with custom properties:
- `type` = `Cancellation`
- `delayMinutes` = `0`

```json
{
  "disruptionId": "DISR-20260407-002",
  "flightNumber": "HV1234",
  "type": "Cancellation",
  "reason": "Crew shortage",
  "passengersAffected": 174
}
```

Verify the message lands in `disruption-priority` queue:

```bash
az servicebus queue show \
  --name disruption-priority \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"
```

### Step 7 — Remove Auto-Forwarding

```bash
az servicebus queue update \
  --name disruption-triage \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --forward-to ""
```

---

## Auto-Forwarding Patterns

| Pattern | Description |
|---------|-------------|
| **Queue chaining** | Queue A → Queue B → Queue C (pipeline) |
| **Topic to Queue** | Subscription with filter → Dedicated queue |
| **DLQ forwarding** | Forward dead-lettered messages to an analysis queue |
| **Fan-in** | Multiple subscriptions → Single processing queue |

---

## Key Takeaways

- Auto-forwarding creates atomic, broker-side chaining (no consumer code needed for forwarding)
- The destination must exist in the same namespace
- Forwarded messages skip the source entity's consumers (message goes straight to destination)
- Combine with filters on subscriptions for intelligent routing
- Auto-forwarding dead-letter queues can aggregate all DLQ messages in one place

---

## Review Questions

1. If `disruption-triage` has both a consumer AND auto-forwarding enabled, what happens?

   > **Answer:** The auto-forwarding takes precedence — messages are forwarded **before** any consumer can receive them. The consumer on `disruption-triage` will **never see any messages**. Auto-forwarding happens at the broker level when the message is enqueued, not at receive time. If you need both a consumer and forwarding, use a Topic with two subscriptions: one for the consumer and one with auto-forwarding.

2. Can you create a circular auto-forwarding chain (A → B → A)? What would happen?

   > **Answer:** Azure Service Bus **prevents circular forwarding** at configuration time. When you try to set up a chain that would create a cycle, the operation fails with an error. The broker validates the forwarding graph to ensure there are no loops. This protects against infinite message loops that would consume resources and fill up queues.

3. How does auto-forwarding interact with message TTL?

   > **Answer:** The message's TTL **continues to count down** as it passes through the forwarding chain. The TTL is set when the message is originally sent and is not reset by forwarding. If a message spends time in transit through multiple hops, it can expire and be dead-lettered at any point in the chain. The effective TTL at the destination is the original TTL minus the time spent in transit.
