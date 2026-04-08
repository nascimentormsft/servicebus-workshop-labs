# Lab 06 – Message Locks and Settlement

## Objective

Understand and experiment with **message lock** and **settlement** mechanisms — demonstrated with a **flight crew assignment** system where assignments must not be processed by multiple schedulers simultaneously.

## Scenario

Transavia's crew scheduling system assigns crew members to flights. When a scheduler picks up an assignment request, no other scheduler should process the same request. Service Bus uses **PeekLock** to ensure exclusive processing: the message is locked to one consumer and must be explicitly settled (Completed, Abandoned, or Dead-lettered).

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| PeekLock Receive | **Supported** | **Supported** | **Supported** |
| ReceiveAndDelete | **Supported** | **Supported** | **Supported** |
| Max Lock Duration | 5 minutes | 5 minutes | 5 minutes |
| Lock Renewal | Supported | Supported | Supported |

> In production crew scheduling, always use **PeekLock** mode. ReceiveAndDelete risks losing messages if the consumer crashes before processing completes.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create a Queue with a Custom Lock Duration

```bash
az servicebus queue create \
  --name crew-assignments \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --lock-duration PT30S \
  --max-delivery-count 3
```

> `--lock-duration PT30S` sets the lock to 30 seconds (short for demo purposes). `--max-delivery-count 3` means after 3 failed deliveries, the message goes to dead-letter.

### Step 2 — Send Crew Assignment Messages

Using Service Bus Explorer, send to `crew-assignments`:

**Message 1:**
```json
{
  "assignmentId": "CA-2026-0407-001",
  "flightNumber": "HV6321",
  "role": "Captain",
  "requiredLicense": "B737-800",
  "date": "2026-05-15",
  "base": "AMS"
}
```

**Message 2:**
```json
{
  "assignmentId": "CA-2026-0407-002",
  "flightNumber": "HV6321",
  "role": "FirstOfficer",
  "requiredLicense": "B737-800",
  "date": "2026-05-15",
  "base": "AMS"
}
```

**Message 3:**
```json
{
  "assignmentId": "CA-2026-0407-003",
  "flightNumber": "HV5102",
  "role": "CabinCrewLead",
  "requiredLicense": "CabinCrew",
  "date": "2026-05-15",
  "base": "AMS"
}
```

### Step 3 — Receive in PeekLock Mode (Do Not Complete)

1. In Service Bus Explorer, switch to **Receive** mode
2. Select **PeekLock** receive mode
3. Click **Receive** — you get Message 1
4. **DO NOT click Complete** — wait and observe

### Step 4 — Observe Lock Behavior

While Message 1 is locked:

1. Open a **second browser tab** to the same Service Bus Explorer
2. Try to **Receive** again — you should get **Message 2** (not Message 1)
3. This proves Message 1 is locked and invisible to other consumers

### Step 5 — Wait for Lock Expiry

1. Go back to the first tab where Message 1 is locked
2. Wait 30 seconds (the lock duration we configured)
3. The lock expires — the message becomes visible again
4. Receive again — Message 1 reappears, and its **DeliveryCount** is incremented

### Step 6 — Complete a Message

1. Receive Message 1 again
2. This time, click **Complete**
3. The message is permanently removed from the queue
4. Check the queue count — it should decrease by 1

### Step 7 — Abandon a Message

1. Receive Message 2
2. Click **Abandon** instead of Complete
3. The message is immediately released back to the queue (lock released)
4. The **DeliveryCount** increments
5. Receive again — Message 2 is immediately available

### Step 8 — Experiment with ReceiveAndDelete

1. Send a new test message to the queue
2. Switch to **ReceiveAndDelete** mode in Service Bus Explorer
3. Receive the message — it is immediately deleted, no Complete/Abandon options
4. Confirm the message is gone from the queue

---

## Settlement Options Summary

| Action | Effect | Use When |
|--------|--------|----------|
| **Complete** | Message is permanently removed | Processing succeeded |
| **Abandon** | Lock released, message returns to queue, DeliveryCount++ | Transient failure, want to retry |
| **Dead-letter** | Message moved to DLQ with reason | Permanent failure, poison message |
| **Defer** | Message stays in queue but can only be retrieved by sequence number | Processing should happen later |

---

## Lock Duration Guidance for Aviation Systems

| System | Recommended Lock Duration | Reasoning |
|--------|---------------------------|-----------|
| Booking confirmation | 30 seconds | Fast processing |
| Crew assignment | 1–2 minutes | May need database lookups |
| Baggage reconciliation | 2–3 minutes | Complex multi-system validation |
| Flight plan filing | 3–5 minutes | External API calls to ATC systems |

> If processing may exceed 5 minutes, use **lock renewal** in your consumer code to extend the lock before it expires.

---

## Review Questions

1. What happens to in-flight messages if the consumer process crashes in PeekLock mode?

   > **Answer:** The messages remain safely in the queue. When the consumer crashes, it cannot Complete or Abandon the locked messages. After the **lock duration expires**, the locks are automatically released, and the messages become visible for redelivery. The `DeliveryCount` increments on next delivery. This is exactly why PeekLock is the recommended mode for mission-critical systems — no messages are lost on consumer failure.

2. Why is ReceiveAndDelete risky for crew assignment processing?

   > **Answer:** In ReceiveAndDelete mode, the message is **permanently deleted the instant it's delivered** to the consumer — before any processing occurs. If the consumer crashes after receiving but before completing the crew assignment logic (e.g., database update, notification), the assignment message is **lost forever**. There's no retry, no dead-letter, no recovery. For crew assignments, losing a message could mean a flight departs without required crew.

3. A message has DeliveryCount = 10 but max-delivery-count is 5. What happened?

   > **Answer:** This shouldn't happen under normal circumstances — the message should have been dead-lettered at count 5. However, `DeliveryCount` can exceed `MaxDeliveryCount` if the `MaxDeliveryCount` was **lowered after** the message was already in the queue with a higher count. Another scenario: the message was explicitly received from the dead-letter queue and resubmitted to the main queue, carrying its original DeliveryCount forward.
