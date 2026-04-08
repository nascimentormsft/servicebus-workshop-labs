# Lab 02 – Service Bus Topics & Subscriptions

## Objective

Create a Service Bus Topic with multiple subscriptions to implement a **flight status notification** fan-out pattern — where a single flight status update must be delivered to multiple downstream systems simultaneously.

## Scenario

When a flight's status changes (e.g., delayed, gate changed, cancelled), multiple systems need to react: the passenger notification system sends SMS/email, the crew management system adjusts crew schedules, the airport display system updates departure boards, and the ground handling system adjusts turnaround plans. A Topic with Subscriptions delivers the same message to all interested parties.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Topics & Subscriptions | **Not supported** | Supported | Supported |
| Max Subscriptions per Topic | — | 2,000 | 2,000 |
| Max Topics per Namespace | — | 10,000 | 10,000 |

> **Important:** Topics are NOT available on the Basic tier. You must use Standard or Premium.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 00:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create a Topic for Flight Status Updates

```bash
az servicebus topic create \
  --name flight-status-updates \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --max-size 1024 \
  --default-message-time-to-live P7D
```

### Step 2 — Create Subscriptions for Each Consumer System

```bash
# Passenger notifications (SMS, email, push)
az servicebus topic subscription create \
  --name passenger-notifications \
  --topic-name flight-status-updates \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live P3D

# Crew management system
az servicebus topic subscription create \
  --name crew-management \
  --topic-name flight-status-updates \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live P3D

# Airport display boards
az servicebus topic subscription create \
  --name display-boards \
  --topic-name flight-status-updates \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live PT6H

# Ground handling operations
az servicebus topic subscription create \
  --name ground-handling \
  --topic-name flight-status-updates \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live P1D
```

> Notice the different TTL values: display boards only need 6 hours of history, while crew management may need 3 days.

### Step 3 — List Subscriptions

```bash
az servicebus topic subscription list \
  --topic-name flight-status-updates \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --output table
```

### Step 4 — Explore in the Azure Portal

1. Navigate to your Service Bus namespace
2. Click on **Topics** → `flight-status-updates`
3. Observe the **Subscription count** on the overview blade
4. Click into each subscription and review its properties

### Step 5 — Send a Flight Status Message via Service Bus Explorer

1. In the portal, go to **Topics** → `flight-status-updates`
2. Open **Service Bus Explorer**
3. Click **Send messages**
4. Set **Content Type** to `application/json`
5. Add the following **Custom Properties** (click "Add property"):
   - `flightNumber` = `HV6321`
   - `statusType` = `Delayed`
   - `airport` = `AMS`
6. Paste the body:

```json
{
  "flightNumber": "HV6321",
  "route": "AMS-BCN",
  "originalDeparture": "2026-05-15T14:30:00Z",
  "newDeparture": "2026-05-15T16:00:00Z",
  "status": "Delayed",
  "reason": "Late incoming aircraft",
  "gate": "D47",
  "terminal": "3"
}
```

7. Click **Send**

### Step 6 — Verify Fan-out Delivery

1. Navigate to each subscription in the Portal
2. Check the **Active message count** — each subscription should show **1 message**
3. Use Service Bus Explorer on the `passenger-notifications` subscription to **Peek** the message
4. Repeat for `crew-management` — confirm both received the same message

### Step 7 — Receive from One Subscription

1. Open Service Bus Explorer on the `display-boards` subscription
2. **Receive** the message (PeekLock mode)
3. **Complete** the message
4. Verify that `display-boards` now shows 0 messages, while other subscriptions still have 1

---

## Key Takeaways

- A **Queue** is point-to-point: one sender, one receiver
- A **Topic** is publish-subscribe: one sender, many receivers via subscriptions
- Each subscription maintains its own independent copy of the message
- Receiving from one subscription does not affect other subscriptions

---

## Review Questions

1. If the booking system only needs one consumer, should you use a Queue or a Topic?

   > **Answer:** Use a **Queue**. Queues are designed for point-to-point communication (one sender, one receiver). They are simpler, require Basic tier or above, and are more efficient when there's only one consumer. Use a Topic only when you need to fan out the same message to multiple independent subscribers.

2. What happens to messages sent to a Topic with zero subscriptions?

   > **Answer:** The messages are **silently discarded**. A Topic without subscriptions has no delivery targets, so messages are accepted by the broker but immediately dropped. This is a common pitfall — always create subscriptions before sending messages, or messages will be lost.

3. Why might the `display-boards` subscription have a shorter TTL than `crew-management`?

   > **Answer:** Display boards show real-time information — a gate change from 6 hours ago is no longer relevant and would confuse passengers if displayed. Crew management, however, may need to reference flight status changes for shift planning, rebooking, and reporting over several days. TTL should match how long the data is operationally useful to the consumer.
