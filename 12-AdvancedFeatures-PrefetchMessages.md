# Lab 12 – Prefetch Messages

## Objective

Understand **message prefetching** and its impact on throughput — demonstrated with a **high-volume baggage tracking** system where thousands of scan events per minute require optimized message consumption.

## Scenario

Transavia's baggage handling system at Schiphol processes barcode scan events from conveyor belts. Each scan generates a message. During peak hours (early morning departures), the system handles 3,000+ scans per minute. Prefetching reduces round-trips to the broker by fetching multiple messages in a single call, dramatically improving throughput.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Prefetch | **Supported** | **Supported** | **Supported** |
| Network Latency Impact | Shared infra | Shared infra | Dedicated, low latency |

> Prefetching benefits Standard tier most, where shared infrastructure introduces variable latency. On Premium tier, dedicated resources already provide low latency, but prefetch still helps by reducing round-trips.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create a Queue for Baggage Scans

```bash
az servicebus queue create \
  --name baggage-scans \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --lock-duration PT1M \
  --default-message-time-to-live P1D
```

### Step 2 — Send Multiple Messages (Simulating Scan Events)

Using Service Bus Explorer, send at least 10 messages. Here's a sample pattern — repeat with different tag numbers:

```json
{
  "scanId": "SCAN-001",
  "baggageTag": "TV0012345",
  "location": "AMS-T3-BELT-A7",
  "timestamp": "2026-04-07T06:15:30Z",
  "flightNumber": "HV6321",
  "scanType": "Departure"
}
```

```json
{
  "scanId": "SCAN-002",
  "baggageTag": "TV0012346",
  "location": "AMS-T3-BELT-A7",
  "timestamp": "2026-04-07T06:15:31Z",
  "flightNumber": "HV6321",
  "scanType": "Departure"
}
```

Continue sending up to SCAN-010 with incrementing IDs and tags.

### Step 3 — Understand Prefetch Conceptually

**Without prefetch (PrefetchCount = 0):**
```
Consumer → [Request Message] → Broker → [Return 1 Message] → Consumer
Consumer → [Request Message] → Broker → [Return 1 Message] → Consumer
Consumer → [Request Message] → Broker → [Return 1 Message] → Consumer
... (10 round-trips for 10 messages)
```

**With prefetch (PrefetchCount = 10):**
```
Consumer → [Request Message] → Broker → [Return 1 + Prefetch 9] → Consumer
Consumer → [Local buffer: 9 messages remaining, no network call]
Consumer → [Local buffer: 8 messages remaining, no network call]
... (1 round-trip for 10 messages)
```

### Step 4 — Observe Prefetch Behavior in the Portal

> Note: Prefetch is a **client-side** configuration, not a queue setting. It's set in the consumer code. Service Bus Explorer does not directly expose prefetch settings, but we can observe its effects.

1. Navigate to the queue in the Portal
2. Check the **Active message count**: should show 10
3. In production, prefetch is configured in code:

```csharp
// .NET SDK example
var client = new ServiceBusClient(connectionString);
var receiver = client.CreateReceiver("baggage-scans", new ServiceBusReceiverOptions
{
    PrefetchCount = 20
});
```

```python
# Python SDK example
from azure.servicebus import ServiceBusClient

client = ServiceBusClient.from_connection_string(conn_str)
receiver = client.get_queue_receiver(
    queue_name="baggage-scans",
    prefetch_count=20
)
```

### Step 5 — Prefetch and Lock Duration Interaction

Here's the critical relationship:

```
Lock starts when message is PREFETCHED (not when your code processes it)
```

**Example with PrefetchCount = 100 and LockDuration = 30s:**
- 100 messages are fetched at once, all locks start simultaneously
- If processing each message takes 500ms, message 60 is processed at ~30s
- Messages 61-100 have **expired locks** → they return to the queue → duplicate processing!

**Calculate safe prefetch count:**
```
Safe PrefetchCount = LockDuration / MessageProcessingTime × SafetyFactor

Example:
LockDuration = 60s
ProcessingTime = 200ms per message
SafetyFactor = 0.8

PrefetchCount = (60 / 0.2) × 0.8 = 240
```

### Step 6 — Verify Lock Duration Impact

1. Receive a message in PeekLock via Service Bus Explorer
2. Wait for the lock to expire (check your queue's lock duration)
3. Peek the queue — the message reappears for redelivery
4. This is what happens to prefetched messages when the consumer is too slow

### Step 7 — Monitor Prefetch Impact

In production, monitor these metrics to tune prefetch:

```bash
# Check message counts and processing metrics
az servicebus queue show \
  --name baggage-scans \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "{active: countDetails.activeMessageCount, deadLetter: countDetails.deadLetterMessageCount, size: sizeInBytes}"
```

---

## Prefetch Guidelines

| Scenario | Recommended PrefetchCount | Reasoning |
|----------|--------------------------|-----------|
| Low volume (< 10 msg/min) | 0 (disabled) | Prefetch adds no benefit |
| Medium volume (10-100 msg/min) | 10-50 | Reduces latency |
| High volume (100+ msg/min) | 50-300 | Maximizes throughput |
| Session-enabled queues | Lower values (5-20) | Each session processes sequentially |

---

## Key Takeaways

- Prefetch is a **client-side optimization**, not a broker feature
- It reduces network round-trips by fetching messages speculatively
- Lock duration must be long enough to cover the prefetch buffer processing time
- Too-high prefetch with short lock = duplicate processing risk
- Monitor `DeliveryCount` to detect prefetch misconfiguration

---

## Review Questions

1. If PrefetchCount = 50 and LockDuration = 30s, what is the maximum safe per-message processing time?

   > **Answer:** Approximately **0.6 seconds** (600ms) per message. Calculation: `LockDuration / PrefetchCount = 30s / 50 = 0.6s`. The last prefetched message (message #50) must be processed before its lock expires. Since all 50 locks start simultaneously at prefetch time, you need to process all 50 within 30 seconds. In practice, apply a safety factor (e.g., 80%): `30s / 50 * 0.8 = 0.48s` per message.

2. Should you use prefetch with ReceiveAndDelete mode? What are the risks?

   > **Answer:** You **can**, but it's risky. In ReceiveAndDelete + prefetch, all prefetched messages are **immediately deleted from the broker** and buffered on the client. If the consumer crashes, all buffered messages are **permanently lost** — there's no lock to expire, no retry, and no dead-letter. For non-critical, high-volume data (e.g., telemetry), this may be acceptable for maximum throughput. For booking or crew messages, never combine prefetch with ReceiveAndDelete.

3. Why might you use a lower PrefetchCount for session-enabled queues?

   > **Answer:** In session-enabled queues, a consumer processes one session at a time sequentially. Prefetching many messages from a single session means all those locks start simultaneously, but they're processed one-by-one. A PrefetchCount of 50 with slow per-message processing could easily exceed the lock duration. Additionally, in session mode, you can't parallelize messages within the same session, so the benefit of a large prefetch buffer is reduced.
