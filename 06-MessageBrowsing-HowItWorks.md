# Lab 06 – Message Browsing (Peek) for Troubleshooting

## Objective

Use **Message Browsing (Peek)** to inspect messages in queues and subscriptions **without consuming them** — a critical troubleshooting technique when investigating why messages are stuck, malformed, or not being processed.

## Scenario

Transavia's ground handling system has stopped processing baggage routing messages. The operations team needs to investigate what messages are in the queue without removing them. Message Browsing lets you "look inside" the queue non-destructively, making it invaluable for production troubleshooting.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Message Browsing (Peek) | **Supported** | **Supported** | **Supported** |

> Peek is available on all tiers. It does NOT acquire a lock and does NOT affect message state — making it safe to use in production at any time.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Set Up the Scenario

Create a queue and populate it with several messages simulating a stuck system:

```bash
az servicebus queue create \
  --name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --max-size 1024
```

### Step 2 — Send Multiple Messages to Simulate a Backlog

Using Service Bus Explorer, send the following messages to `baggage-routing`:

**Message 1 — Normal message:**
```json
{
  "baggageTag": "TV0001001",
  "flightNumber": "HV6321",
  "passenger": "Jan de Vries",
  "route": "AMS-BCN",
  "weight": 18.5,
  "priority": false
}
```

**Message 2 — Malformed message (missing route — this causes processing failure):**
```json
{
  "baggageTag": "TV0001002",
  "flightNumber": "HV5102",
  "passenger": "Anna Kuijpers",
  "route": null,
  "weight": 23.0,
  "priority": false
}
```

**Message 3 — Priority baggage:**
```json
{
  "baggageTag": "TV0001003",
  "flightNumber": "HV6321",
  "passenger": "Carlos Martinez",
  "route": "AMS-BCN",
  "weight": 12.0,
  "priority": true
}
```

**Message 4 — Another normal message:**
```json
{
  "baggageTag": "TV0001004",
  "flightNumber": "HV9988",
  "passenger": "Sophie Laurent",
  "route": "AMS-CDG",
  "weight": 20.0,
  "priority": false
}
```

### Step 3 — Browse Messages Using Service Bus Explorer (Portal)

1. Navigate to **Queues** → `baggage-routing`
2. Open **Service Bus Explorer**
3. Select **Peek** mode (this is the default)
4. Click **Peek from start**
5. Observe:
   - Messages are returned in enqueue order
   - Each message shows: **Sequence Number**, **Message ID**, **Enqueued Time**, and **Body**
   - The **Active message count** does NOT change after peeking

### Step 4 — Examine Individual Message Properties

1. Click on **Message 2** in the peeked results
2. Inspect the **Properties** tab:
   - `SequenceNumber` — unique, monotonically increasing number assigned by the broker
   - `EnqueuedTimeUtc` — when the message was received by Service Bus
   - `MessageId` — set by the sender (or auto-generated)
   - `DeliveryCount` — how many times this message has been delivered (0 if never received)
   - `TimeToLive` — remaining TTL
3. Inspect the **Body** tab — find the malformed data (`"route": null`)

### Step 5 — Peek Again and Confirm Non-Destructive

1. Click **Peek from start** again
2. You should see the **same 4 messages** — peek never removes messages
3. Check the queue message count:

```bash
az servicebus queue show \
  --name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount"
```

The count should still be `4`.

### Step 6 — Peek by Sequence Number

You can peek starting from a specific sequence number to skip already-inspected messages:

1. Note the **Sequence Number** of Message 3
2. In Service Bus Explorer, use "Peek from sequence number" with that value
3. Only Message 3 and 4 should appear

### Step 7 — Compare Peek vs. Receive Behavior

1. Switch to **Receive** mode
2. Select **PeekLock** receive mode
3. Receive one message
4. Observe: the message is now **locked** (temporarily invisible to other receivers)
5. Click **Complete** to permanently remove it
6. Switch back to **Peek** mode — you should now see 3 messages

---

## Troubleshooting Workflow Using Peek

When a consumer stops processing messages, use this investigation sequence:

```
1. Peek the queue → Are there messages? (If not, the issue is on the sender side)
2. Inspect message bodies → Are messages malformed?
3. Check DeliveryCount → Has the same message been retried many times?
4. Check EnqueuedTimeUtc → When did the backlog start?
5. Peek the Dead Letter Queue → Are messages being dead-lettered?
```

### Step 8 — Peek the Dead Letter Queue

1. In the Portal, navigate to the queue's overview
2. Note the **Dead-letter message count**
3. In Service Bus Explorer, switch to the **Dead letter** sub-queue tab
4. Peek — any messages here were moved due to max delivery attempts or expiration

```bash
# CLI: Check dead-letter count
az servicebus queue show \
  --name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.deadLetterMessageCount"
```

---

## Key Takeaways

- **Peek** reads messages without locking or removing them — completely non-destructive
- Peek returns messages in sequence order, starting from where you specify
- In production, Peek is your first tool when investigating queue issues
- Always check both the **active queue** and the **dead-letter queue** during troubleshooting
- Peek works on queues, topic subscriptions, and their dead-letter sub-queues

---

## Review Questions

1. A queue shows 500 active messages but the consumer says it's receiving nothing. What would you check first using Peek?

   > **Answer:** Peek the messages and inspect:
   > - **DeliveryCount**: If it's high on all messages, the consumer IS receiving them but failing to process (likely crashing or abandoning). The messages keep returning to the queue.
   > - **Message body/properties**: Are the messages malformed or incompatible with the consumer's current code version?
   > - **Session ID**: If the queue is session-enabled, the consumer must specify a session to receive. If the consumer isn't session-aware, it gets nothing.
   > - Also check the consumer's connection string, queue name, and whether it's connecting to the right namespace.

2. What is the difference between `Peek` and `Receive` in PeekLock mode?

   > **Answer:** **Peek** reads the message without any lock or state change — the message remains fully available to consumers. **Receive in PeekLock mode** acquires an exclusive lock on the message, making it temporarily invisible to other receivers. The receiver must then Complete, Abandon, Dead-letter, or Defer the message before the lock expires. If the lock expires without settlement, the message becomes visible again.

3. Can Peek interfere with a running consumer or cause message loss?

   > **Answer:** **No.** Peek is completely non-destructive and does not acquire locks. It has no effect on message state, delivery count, or visibility. You can safely Peek a production queue while consumers are actively processing without any impact. This makes it the ideal first tool for troubleshooting.
