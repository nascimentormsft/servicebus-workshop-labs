# Lab 10 – Dead-Letter Queues

## Objective

Understand and work with **Dead-Letter Queues (DLQ)** — the built-in quarantine mechanism for messages that cannot be processed — demonstrated with a **flight plan validation** system where invalid flight plans are automatically isolated.

## Scenario

Transavia's automated flight plan system receives flight plans from dispatch. If a plan has invalid data (missing waypoints, incorrect fuel calculations), the processor rejects it. After multiple retry attempts, the message is moved to the Dead-Letter Queue where an operations engineer can review it manually. The DLQ is also a safety net for expired messages and messages that exceed max delivery count.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Dead-Letter Queue | **Supported** | **Supported** | **Supported** |
| DLQ per Queue | Auto-created | Auto-created | Auto-created |
| DLQ per Subscription | N/A | Auto-created | Auto-created |

> Every queue and subscription has a DLQ automatically. No extra cost applies to DLQ messages, but they DO count against the namespace storage quota.

---

## Exercise Steps

### Step 1 — Create a Queue with Dead-Lettering Options

```bash
az servicebus queue create \
  --name flight-plan-validation \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --max-delivery-count 3 \
  --enable-dead-lettering-on-message-expiration true \
  --default-message-time-to-live PT5M \
  --lock-duration PT30S
```

### Step 2 — Send a "Poison" Message (Simulating Invalid Data)

Using Service Bus Explorer, send to `flight-plan-validation`:

```json
{
  "flightPlanId": "FP-HV6321-20260515",
  "flightNumber": "HV6321",
  "route": "AMS-BCN",
  "waypoints": [],
  "fuelRequired": -500,
  "status": "Invalid",
  "error": "Missing waypoints and negative fuel value"
}
```

### Step 3 — Simulate Consumer Failure (Repeated Abandons)

1. **Receive** the message in PeekLock mode
2. Click **Abandon** (simulates processing failure) — DeliveryCount becomes 1
3. **Receive** again → **Abandon** — DeliveryCount becomes 2
4. **Receive** again → **Abandon** — DeliveryCount becomes 3
5. The message has now reached `max-delivery-count` (3)
6. The next delivery attempt automatically moves it to the DLQ

### Step 4 — Verify the Message is in the DLQ

```bash
az servicebus queue show \
  --name flight-plan-validation \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --query "{active: countDetails.activeMessageCount, deadLetter: countDetails.deadLetterMessageCount}"
```

Expected: `active: 0, deadLetter: 1`

### Step 5 — Inspect the Dead-Lettered Message

1. In Service Bus Explorer, switch to the **Dead letter** tab
2. **Peek** the message
3. Examine the system properties:
   - `DeadLetterReason`: `MaxDeliveryCountExceeded`
   - `DeadLetterErrorDescription`: explains why it was dead-lettered
   - `DeliveryCount`: shows the final count

### Step 6 — Test TTL-Based Dead-Lettering

1. Send a new message with a **per-message TTL of 30 seconds**:

```json
{
  "flightPlanId": "FP-HV5102-20260515",
  "flightNumber": "HV5102",
  "route": "AMS-CDG",
  "waypoints": ["EHRD", "LFPG"],
  "fuelRequired": 12000,
  "status": "Pending"
}
```

2. Wait 30 seconds without receiving it
3. Peek the active queue — message should be gone
4. Check the DLQ — the expired message should appear with:
   - `DeadLetterReason`: `TTLExpiredException`

### Step 7 — Explicitly Dead-Letter a Message

1. Send a new message to the queue
2. **Receive** it in PeekLock mode
3. Instead of Complete or Abandon, click **Dead-letter**
4. You can optionally provide a reason and description
5. The message is immediately moved to the DLQ with your custom reason

### Step 8 — Process (Resubmit) Messages from the DLQ

Messages in the DLQ can be manually inspected and resubmitted:

1. **Peek** a dead-lettered message to understand the issue
2. Copy the message body
3. Fix the data (e.g., add waypoints, correct fuel value)
4. **Send** the corrected message back to the main queue
5. **Receive and Complete** the message from the DLQ to clean it up

### Step 9 — Monitor DLQ Growth

```bash
# Check DLQ count for all queues in the namespace
az servicebus queue list \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --query "[].{name: name, active: countDetails.activeMessageCount, deadLetter: countDetails.deadLetterMessageCount}" \
  --output table
```

---

## Dead-Letter Reasons Summary

| Reason | Trigger | Example |
|--------|---------|---------|
| `MaxDeliveryCountExceeded` | Message delivered more times than `maxDeliveryCount` | Poison message causing consumer crash |
| `TTLExpiredException` | Message TTL expired while in queue | Gate notification delivered too late |
| `HeaderSizeExceeded` | Message headers exceed limit | Too many custom properties |
| Custom reason | Application explicitly dead-letters | Business validation failure |

---

## DLQ Best Practices

1. **Monitor DLQ counts** — a growing DLQ signals consumer issues
2. **Set alerts** on `DeadLetteredMessages` metric (covered in Lab 20)
3. **Build DLQ processors** for automated resubmission of fixable issues
4. **DLQ has no TTL** — messages stay forever unless explicitly removed
5. **DLQ has a max size** — it shares the queue's max size quota

---

## Review Questions

1. What is the DLQ path for a queue named `payment-transactions`? (Hint: `<queuename>/$deadletterqueue`)
2. Why don't DLQ messages have a TTL?
3. Should you auto-resubmit all DLQ messages? What risks are involved?
