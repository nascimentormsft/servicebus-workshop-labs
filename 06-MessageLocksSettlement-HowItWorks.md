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

### Step 1 — Create a Queue with Lock and Delivery Settings

```bash
az servicebus queue create \
  --name crew-assignments \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --lock-duration PT30S \
  --max-delivery-count 3
```

> `--lock-duration PT30S` sets the lock to 30 seconds. `--max-delivery-count 3` means after 3 failed deliveries, the message goes to dead-letter. These settings matter when consuming via SDK with PeekLock mode.

### Step 2 — Verify Lock Settings in the Portal

1. Navigate to **Queues** → `crew-assignments`
2. In the **Overview** blade, note the **Lock duration** (30 seconds) and **Max delivery count** (3)
3. These settings control how PeekLock behaves when applications consume messages via SDK

### Step 3 — Send Crew Assignment Messages

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

### Step 4 — Peek Messages (Non-Destructive)

1. In Service Bus Explorer, switch to **Peek** mode
2. Click **Peek from start** — you see all 3 messages
3. Note: the messages are still in the queue — peeking does not remove or lock them

Verify the count has not changed:

```bash
az servicebus queue show \
  --name crew-assignments \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"
```

> **Peek** is a read-only operation. It does not acquire a lock, does not increment `DeliveryCount`, and does not remove the message. This is what Service Bus Explorer uses for browsing.

### Step 5 — Receive in PeekLock Mode and Complete

1. Switch to **Receive** mode in Service Bus Explorer
2. Click **Receive messages**
3. In Settings, select **PeekLock** and click **Receive**
4. Message 1 appears in the list — it is now **locked**
5. **Select the message** (click on it) — the ribbon reveals settlement options: **Complete**, **Abandon**, **Dead-letter**, and **Defer**
6. Click **Complete** — the message is permanently removed from the queue

> **⚠ WARNING — Do NOT click Defer during this lab.** A deferred message is removed from the normal receive flow and can **only** be retrieved by its sequence number via SDK code. The Portal's Service Bus Explorer **cannot retrieve deferred messages**. Deferred messages still count toward `activeMessageCount`, so you'll see messages in the count but won't be able to receive them. If you accidentally defer a message, you'll need to use the SDK or delete and recreate the queue.

Verify the count decreased:

```bash
az servicebus queue show \
  --name crew-assignments \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"
```

### Step 6 — Receive in PeekLock Mode and Abandon

1. Click **Receive messages** again (still in PeekLock mode)
2. Message 2 appears — select it
3. Click **Abandon** — the lock is released and the message returns to the queue
4. **Peek** the queue — Message 2 is still there, and its **DeliveryCount** has incremented

> **Abandon** simulates a transient failure: the consumer couldn't process the message, so it releases it back for another attempt.

### Step 7 — Observe Lock Expiry

1. Click **Receive messages** in PeekLock mode again
2. Message 2 appears — **do not select it, do not click Complete or Abandon**
3. Wait for the lock duration (30 seconds) — the status changes to **Lock expired**
4. **Peek** the queue — Message 2 is still there, available for redelivery

> **This is the key insight for aviation systems:** If a crew scheduling service crashes mid-processing (equivalent to letting the lock expire), the assignment message is NOT lost — it reappears after the lock expires and is picked up by another instance.

### Step 8 — Receive with ReceiveAndDelete

1. Click **Receive messages**
2. In Settings, switch to **ReceiveAndDelete** and click **Receive**
3. Message 2 is received and **immediately deleted** — no settlement options appear
4. The message is gone permanently, regardless of whether processing succeeded

> **ReceiveAndDelete** is faster but risky — if the consumer crashes before processing, the message is lost forever. Use only for non-critical or idempotent workloads.

### Step 9 — Experiment: Peek vs PeekLock vs ReceiveAndDelete

1. **Peek** the remaining message (Message 3) — confirm it's still in the queue
2. **Receive** in **PeekLock** — select the message, then click **Complete** to remove it
3. **Peek** again — the queue is now empty

Summary of what you observed:

| Action | Message Removed? | Settlement Options? | DeliveryCount Changed? |
|---|---|---|---|
| **Peek** | No | None | No |
| **Receive (PeekLock)** → Complete | Yes | Complete, Abandon, Dead-letter | Yes (on delivery) |
| **Receive (PeekLock)** → Abandon | No (returns to queue) | Complete, Abandon, Dead-letter | Yes (+1) |
| **Receive (PeekLock)** → Lock expired | No (returns to queue) | None (lock lost) | Check via Peek |
| **Receive (PeekLock)** → Defer | No (stays as active count) | N/A | N/A |
| **Receive (ReceiveAndDelete)** | Yes (immediately) | None | N/A (message gone) |

> **⚠ About Defer:** Deferred messages are a trap in the Portal. They count as "active" but cannot be received through normal means — only via SDK using the sequence number. If your active message count is higher than what you can receive, you likely have deferred messages. To clean up, delete and recreate the queue.

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
