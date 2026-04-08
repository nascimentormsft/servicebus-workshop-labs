# Lab 10 – Delaying Message Processing with Deferral

## Objective

Use **message deferral** to postpone processing of messages that arrive out of sequence — demonstrated with a **multi-step aircraft turnaround** process where tasks must be completed in a specific order.

## Scenario

Aircraft turnaround involves a sequence: cabin cleaning → catering → fueling → safety inspection → boarding. If the fueling notification arrives before the catering notification, the ground handler defers it — the message stays in the queue but is only retrievable by its sequence number when the system is ready to process it.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Message Deferral | **Supported** | **Supported** | **Supported** |

> Deferral is available on all tiers. Deferred messages count towards the queue's active storage quota.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 00:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create a Queue for Turnaround Tasks

```bash
az servicebus queue create \
  --name aircraft-turnaround \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --lock-duration PT1M \
  --default-message-time-to-live P1D
```

### Step 2 — Send Out-of-Order Turnaround Messages

Using Service Bus Explorer, send these messages **in this order** (intentionally out of the correct processing sequence):

**Message 1 (sent first — but should be processed third):**
```json
{
  "turnaroundId": "TA-HV6321-20260515",
  "aircraft": "PH-HSA",
  "flightNumber": "HV6321",
  "step": 3,
  "task": "Fueling",
  "status": "Ready",
  "estimatedDuration": "25min"
}
```

**Message 2 (sent second — should be processed first):**
```json
{
  "turnaroundId": "TA-HV6321-20260515",
  "aircraft": "PH-HSA",
  "flightNumber": "HV6321",
  "step": 1,
  "task": "CabinCleaning",
  "status": "Ready",
  "estimatedDuration": "15min"
}
```

**Message 3 (sent third — should be processed second):**
```json
{
  "turnaroundId": "TA-HV6321-20260515",
  "aircraft": "PH-HSA",
  "flightNumber": "HV6321",
  "step": 2,
  "task": "Catering",
  "status": "Ready",
  "estimatedDuration": "20min"
}
```

### Step 3 — Receive and Defer Out-of-Order Messages

1. **Receive** a message in PeekLock mode
2. If you get the **Fueling** message (step 3) first:
   - Note its **Sequence Number** (e.g., `1`)
   - Click **Defer** — the message is deferred
   - It's no longer available through normal receive but is NOT deleted
3. **Receive** again — you should get **CabinCleaning** (step 1)
   - This is the correct next step — click **Complete**
4. **Receive** again — you should get **Catering** (step 2)
   - Click **Complete**
5. Now you're ready for step 3 — but it's deferred

### Step 4 — Observe the Deferred Message

> **⚠ Portal Limitation:** The Service Bus Explorer in the Portal **cannot retrieve deferred messages by sequence number**. This is an SDK-only operation (`ReceiveDeferredMessageAsync(sequenceNumber)`). However, you can still **observe** deferred messages.

1. **Peek from start** — the deferred Fueling message IS visible with its state showing as **Deferred**
2. Try **Receive** in PeekLock — the deferred message does **NOT** appear (it's skipped)
3. Check the queue counts:

```bash
az servicebus queue show \
  --name aircraft-turnaround \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "{active: countDetails.activeMessageCount, deadLetter: countDetails.deadLetterMessageCount}"
```

> The deferred message still counts toward `activeMessageCount`, but it is invisible to normal Receive operations. This is the same behavior you saw in Lab 06 — a discrepancy between the active count and what you can actually receive means deferred messages are present.

In production, retrieving this message would look like this (C# SDK):

```csharp
// sequenceNumber was stored when the message was deferred
ServiceBusReceivedMessage deferredMsg = 
    await receiver.ReceiveDeferredMessageAsync(sequenceNumber);
await receiver.CompleteMessageAsync(deferredMsg);
```

### Step 5 — Observe Queue State with Deferred Messages

```bash
az servicebus queue show \
  --name aircraft-turnaround \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "{active: countDetails.activeMessageCount, scheduled: countDetails.scheduledMessageCount, transfer: countDetails.transferMessageCount}"
```

> Deferred messages are no longer in the "active" count but remain in the queue. They can only be retrieved by explicit sequence number lookup.

### Step 6 — Understand the Deferral Lifecycle

Send a new message, defer it, and track it:

1. Send a test message → Peek → note sequence number
2. Receive in PeekLock → Defer
3. Try normal Receive → the deferred message does NOT appear
4. Peek from start → the deferred message IS visible with state "Deferred"
5. Receive by sequence number → successfully retrieved

---

## Deferral vs. Other Approaches

| Approach | Behavior | Use Case |
|----------|----------|----------|
| **Defer** | Message stays in queue, only retrievable by sequence number | Out-of-order processing, workflow coordination |
| **Abandon** | Message returns to queue for immediate redelivery | Transient failures |
| **Scheduled Messages** | Message is invisible until scheduled enqueue time | Known future processing time |

---

## Important Considerations

1. **Tracking sequence numbers is YOUR responsibility** — store them in a database or state store
2. Deferred messages are **not auto-redelivered** — if you lose the sequence number, the message is effectively stuck
3. Deferred messages **still have TTL** — they expire and dead-letter if the TTL elapses
4. Deferral is ideal for **sagas and orchestration patterns** where steps arrive out of order
5. Unlike abandoned messages, deferred messages do NOT increment `DeliveryCount`

---

## Review Questions

1. What happens to a deferred message if its TTL expires?

   > **Answer:** The deferred message is **dead-lettered** (if dead-lettering on expiration is enabled) or **silently discarded**. Deferral does not pause the TTL clock — the message continues to age. If you defer a message with 5 minutes of TTL remaining and don't retrieve it within that time, it expires. This is important to account for in production: ensure the TTL is long enough to cover the expected deferral duration.

2. How would you track sequence numbers of deferred messages in a production system?

   > **Answer:** Store the sequence numbers in a durable external store such as:
   > - **Azure Table Storage** or **Cosmos DB**: key = session/workflow ID, value = list of deferred sequence numbers
   > - **Redis Cache**: for fast lookup and automatic expiry
   > - **A database column** on the workflow/saga entity
   > The key principle: sequence numbers must be persisted outside the consumer process, because if the consumer restarts, in-memory sequence numbers are lost and the deferred messages become unretrievable.

3. Why is deferral preferable to abandon+retry for handling out-of-order messages?

   > **Answer:** **Abandon** returns the message to the queue head and increments `DeliveryCount`. After repeated abandons, it hits `MaxDeliveryCount` and gets dead-lettered — even though the message itself is perfectly valid, just out of order. **Deferral** parks the message without incrementing `DeliveryCount`, keeping it available indefinitely for retrieval by sequence number. It's a deliberate "I'll come back for this" vs. abandon's "I failed, try again."
