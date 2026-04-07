# Lab 08 – Setting TTL: Per-Message Basis

## Objective

Set **Time-to-Live (TTL) on individual messages** to control how long specific messages remain valid — demonstrated with **real-time gate change notifications** that become irrelevant after a flight departs.

## Scenario

Gate change notifications are only useful until the flight departs. A gate change for a flight departing in 30 minutes should expire after 30 minutes — there's no point processing it after the passengers have boarded. Per-message TTL allows the sender to set expiration based on business context, overriding the queue default.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Per-message TTL | **Supported** | **Supported** | **Supported** |
| Message expiration to DLQ | Configurable | Configurable | Configurable |

> When expired messages are important for auditing (e.g., missed notifications), enable **dead-lettering on expiration** to capture them instead of silently discarding.

---

## Exercise Steps

### Step 1 — Create a Queue with a Long Default TTL

```bash
az servicebus queue create \
  --name gate-notifications \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --default-message-time-to-live P7D \
  --enable-dead-lettering-on-message-expiration true
```

> The queue default is 7 days, but individual messages can override this with a shorter TTL.

### Step 2 — Send a Message with a Short Per-Message TTL

Using Service Bus Explorer:

1. Go to **Queues** → `gate-notifications` → **Service Bus Explorer**
2. Click **Send messages**
3. Set **Content Type** to `application/json`
4. Set **Time to Live** to `60` seconds (or `PT1M` depending on the UI)
5. Body:

```json
{
  "notificationId": "GN-001",
  "flightNumber": "HV6321",
  "type": "GateChange",
  "oldGate": "D47",
  "newGate": "E12",
  "departureTime": "2026-04-07T14:30:00Z",
  "message": "Gate changed from D47 to E12 for flight HV6321"
}
```

6. Click **Send**

### Step 3 — Send a Message WITHOUT Per-Message TTL (Uses Queue Default)

1. Send another message **without** setting the Time to Live field:

```json
{
  "notificationId": "GN-002",
  "flightNumber": "HV5102",
  "type": "BoardingCall",
  "gate": "D47",
  "departureTime": "2026-04-07T16:00:00Z",
  "message": "Boarding now open for flight HV5102 at gate D47"
}
```

This message will use the queue's default TTL of 7 days.

### Step 4 — Peek and Compare TTL Values

1. **Peek** both messages
2. Click on each and examine the properties:
   - **Message 1** (GN-001): `TimeToLive` should show ~60 seconds
   - **Message 2** (GN-002): `TimeToLive` should show ~7 days

### Step 5 — Wait for Message 1 to Expire

1. Wait approximately 60 seconds
2. **Peek** the active queue again — Message 1 (GN-001) should be gone
3. Check the **Dead letter** queue tab — Message 1 should appear there (because we enabled dead-lettering on expiration)
4. Verify via CLI:

```bash
az servicebus queue show \
  --name gate-notifications \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --query "{active: countDetails.activeMessageCount, deadLetter: countDetails.deadLetterMessageCount}"
```

Expected: `active: 1, deadLetter: 1`

### Step 6 — Inspect the Dead-Lettered Expired Message

1. In Service Bus Explorer, go to the **Dead letter** tab
2. **Peek** the dead-lettered message
3. Look at the system properties:
   - `DeadLetterReason`: `TTLExpiredException`
   - `DeadLetterErrorDescription`: describes the expiration

### Step 7 — Send Messages with Varying TTLs

Send 3 more messages with different TTLs to observe staggered expiration:

| Message | TTL | Scenario |
|---------|-----|----------|
| `GN-003` | 30 seconds | Immediate gate change — flight boarding now |
| `GN-004` | 5 minutes | Gate change — flight in 1 hour |
| `GN-005` | No per-message TTL | General announcement — use queue default |

Peek periodically and watch them expire one by one.

---

## Per-Message TTL Rules

1. If per-message TTL is set and is **less than** the queue default → message uses the per-message TTL
2. If per-message TTL is set and is **greater than** the queue default → message uses the **queue default** (queue wins)
3. If no per-message TTL is set → message uses the queue default
4. Message expiration is checked at dequeue time, not continuously

> **Important:** The queue's default TTL acts as a **ceiling**. You cannot set a per-message TTL that exceeds it. The effective TTL is always `min(messageTTL, queueTTL)`.

---

## Review Questions

1. A gate notification has a 2-hour TTL but the queue default is 30 minutes. What is the effective TTL?
2. Why is dead-lettering on expiration useful for gate notifications?
3. Should boarding pass messages have a short or long TTL? Why?
