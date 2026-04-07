# Lab 03 – Sessions: Ordered Message Processing

## Objective

Use Service Bus **Sessions** to guarantee ordered processing of messages that belong together — demonstrated with a **passenger check-in workflow** where each step must be processed in sequence.

## Scenario

A passenger's check-in process involves multiple steps that must happen in order: identity verification → seat assignment → baggage tag generation → boarding pass issuance. If these steps are processed out of order, the check-in fails. Sessions group related messages and ensure FIFO processing within each group, while allowing parallel processing across different passengers.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Sessions | Not supported | **Supported** | **Supported** |

> Sessions are available on Standard and Premium tiers. For high-throughput scenarios (thousands of concurrent check-ins), Premium provides dedicated resources to avoid noisy-neighbor effects.

---

## Exercise Steps

### Step 1 — Create a Session-Enabled Queue

```bash
az servicebus queue create \
  --name checkin-workflow \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --enable-session true \
  --lock-duration PT1M
```

> **Note:** Once a queue is created with sessions enabled, this setting CANNOT be changed. You would need to delete and recreate the queue.

### Step 2 — Verify Session Support in the Portal

1. Navigate to **Queues** → `checkin-workflow`
2. In the **Overview** blade, confirm **Requires session** is `true`

### Step 3 — Send Ordered Messages for Passenger A

Using Service Bus Explorer in the Portal:

1. Go to **Queues** → `checkin-workflow` → **Service Bus Explorer**
2. Click **Send messages**
3. Send the following 4 messages **one at a time**, each with:
   - **Session ID**: `PAX-JDV-HV6321` (same for all four)
   - **Content Type**: `application/json`

**Message 1:**
```json
{
  "step": 1,
  "action": "IdentityVerification",
  "passengerId": "PAX-JDV",
  "flightNumber": "HV6321",
  "documentType": "Passport",
  "documentNumber": "NP1234567"
}
```

**Message 2:**
```json
{
  "step": 2,
  "action": "SeatAssignment",
  "passengerId": "PAX-JDV",
  "flightNumber": "HV6321",
  "seat": "14A",
  "class": "Economy"
}
```

**Message 3:**
```json
{
  "step": 3,
  "action": "BaggageTagGeneration",
  "passengerId": "PAX-JDV",
  "flightNumber": "HV6321",
  "baggageCount": 1,
  "tagNumber": "TV0987654"
}
```

**Message 4:**
```json
{
  "step": 4,
  "action": "BoardingPassIssued",
  "passengerId": "PAX-JDV",
  "flightNumber": "HV6321",
  "gate": "D47",
  "boardingGroup": "3"
}
```

### Step 4 — Send Messages for Passenger B (Different Session)

Repeat the send process but with **Session ID**: `PAX-AK-HV5102`

**Message 1:**
```json
{
  "step": 1,
  "action": "IdentityVerification",
  "passengerId": "PAX-AK",
  "flightNumber": "HV5102",
  "documentType": "IDCard",
  "documentNumber": "ID9876543"
}
```

**Message 2:**
```json
{
  "step": 2,
  "action": "SeatAssignment",
  "passengerId": "PAX-AK",
  "flightNumber": "HV5102",
  "seat": "22C",
  "class": "Economy"
}
```

### Step 5 — Peek and Observe Session Ordering

1. In Service Bus Explorer, switch to **Peek** mode
2. Peek messages — note that messages are grouped by Session ID
3. Within each session, messages maintain their send order (step 1, 2, 3, 4)

### Step 6 — Receive Messages by Session

1. Switch to **Receive** mode
2. Enter **Session ID**: `PAX-JDV-HV6321`
3. Receive messages — they arrive in order (step 1 → 2 → 3 → 4)
4. Now enter **Session ID**: `PAX-AK-HV5102` and receive those messages

> **Key insight:** A session-aware receiver locks an entire session. While one consumer processes Passenger A's check-in flow, another consumer can simultaneously process Passenger B's flow — enabling parallelism with guaranteed per-passenger ordering.

---

## Key Takeaways

- Sessions guarantee FIFO within a group (session) while enabling parallelism across groups
- The **Session ID** is a property on the message set by the sender
- A session-enabled queue REQUIRES a Session ID on every message (sending without one will fail)
- Sessions are ideal for workflows, sagas, and ordered processing per entity

---

## Review Questions

1. What happens if you try to send a message without a Session ID to a session-enabled queue?
2. Can two consumers process messages from the same session simultaneously?
3. Why are sessions a better fit than a single queue with ordering for check-in workflows handling hundreds of passengers concurrently?
