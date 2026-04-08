# Lab 08 – Setting TTL: Queue and Topic Level

## Objective

Configure **default Time-to-Live at the queue and topic/subscription level** to enforce organization-wide message retention policies — demonstrated with different aviation messaging categories that have different retention requirements.

## Scenario

Transavia needs different message retention policies for different systems:
- **Operational alerts** (delays, cancellations): relevant for 24 hours
- **Maintenance work orders**: relevant for 30 days
- **Real-time telemetry data** (fuel, weather): relevant for 1 hour

Setting TTL at the queue/topic level establishes a baseline policy that applies to all messages unless overridden.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Queue-level TTL | **Supported** | **Supported** | **Supported** |
| Topic/Subscription-level TTL | N/A | **Supported** | **Supported** |
| Max TTL | Unlimited* | Unlimited* | Unlimited* |

> *The maximum TTL value is `TimeSpan.MaxValue` (~10,675,199 days). For Premium tier with large message volumes, shorter TTLs help manage namespace storage.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 00:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create Queues with Different TTL Policies

```bash
# Operational alerts — 24-hour retention
az servicebus queue create \
  --name operational-alerts \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live P1D \
  --enable-dead-lettering-on-message-expiration true

# Maintenance work orders — 30-day retention
az servicebus queue create \
  --name maintenance-workorders \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live P30D \
  --enable-dead-lettering-on-message-expiration true

# Real-time telemetry — 1-hour retention
az servicebus queue create \
  --name telemetry-data \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live PT1H \
  --enable-dead-lettering-on-message-expiration false
```

> Notice telemetry has dead-lettering **disabled** — expired sensor data is not worth keeping.

### Step 2 — Compare TTL Settings in the Portal

1. Navigate to each queue and compare the **Default message time to live** property:

| Queue | Default TTL | Dead-Letter on Expiry |
|-------|-------------|----------------------|
| `operational-alerts` | 1 day | Yes |
| `maintenance-workorders` | 30 days | Yes |
| `telemetry-data` | 1 hour | No |

### Step 3 — Create a Topic with Subscription-Level TTL

```bash
# Create topic for flight data distribution
az servicebus topic create \
  --name flight-data \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live P7D
```

Create subscriptions with different TTLs:

```bash
# Dashboard subscription — only needs recent data (2 hours)
az servicebus topic subscription create \
  --name dashboard \
  --topic-name flight-data \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live PT2H

# Analytics subscription — needs longer retention (7 days, inherits from topic)
az servicebus topic subscription create \
  --name analytics \
  --topic-name flight-data \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live P7D

# Compliance subscription — needs maximum retention (30 days)
az servicebus topic subscription create \
  --name compliance \
  --topic-name flight-data \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live P30D
```

### Step 4 — Send a Message to the Topic

Using Service Bus Explorer on `flight-data`:

```json
{
  "flightNumber": "HV6321",
  "timestamp": "2026-04-07T14:00:00Z",
  "fuelLevel": 85,
  "altitude": 35000,
  "speed": 480,
  "position": {
    "lat": 48.8566,
    "lon": 2.3522
  }
}
```

### Step 5 — Compare Message TTL Across Subscriptions

Peek the message in each subscription and compare the effective TTL:

| Subscription | Subscription TTL | Topic TTL | Effective TTL |
|---|---|---|---|
| `dashboard` | 2 hours | 7 days | **2 hours** (min wins) |
| `analytics` | 7 days | 7 days | **7 days** |
| `compliance` | 30 days | 7 days | **7 days** (topic ceiling applies) |

> **Key insight:** The effective TTL is the minimum of the message TTL, subscription TTL, and topic TTL. The topic acts as a ceiling for all subscriptions.

### Step 6 — Update TTL on an Existing Queue

```bash
az servicebus queue update \
  --name operational-alerts \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --default-message-time-to-live PT12H
```

> Unlike sessions and duplicate detection, **TTL can be changed after creation**. The new TTL applies to new messages; existing messages keep their original TTL.

### Step 7 — Verify the Updated TTL

```bash
az servicebus queue show \
  --name operational-alerts \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "defaultMessageTimeToLive"
```

---

## TTL Hierarchy and Precedence

```
Effective TTL = min(Per-Message TTL, Entity Default TTL)

For Topics:
Effective TTL = min(Per-Message TTL, Subscription TTL, Topic TTL)
```

| Level | Configurable | Can Exceed Parent? |
|-------|---|---|
| Namespace | No TTL setting | — |
| Queue | Yes (`defaultMessageTimeToLive`) | — |
| Topic | Yes | — |
| Subscription | Yes | No (Topic TTL is the ceiling) |
| Per-Message | Yes | No (Entity TTL is the ceiling) |

---

## Storage Impact

| TTL | Impact |
|-----|--------|
| Very short (< 1 hour) | Low storage usage; good for real-time data |
| Medium (1–7 days) | Moderate storage; good for operational messages |
| Long (> 30 days) | High storage; consider archiving to Azure Storage |

> **Premium consideration:** Premium tier namespaces have 1-4 TB capacity. Monitor namespace storage usage via Azure Monitor when using long TTLs with high message volumes.

---

## Review Questions

1. If a topic has TTL of 2 days and a subscription has TTL of 5 days, what is the effective TTL for messages in that subscription?

   > **Answer:** **2 days.** The topic TTL acts as a ceiling for all its subscriptions. The effective TTL is `min(messageTTL, subscriptionTTL, topicTTL)`. A subscription cannot retain messages longer than the topic allows, even if its own TTL is configured to be longer.

2. Does changing the queue's default TTL affect messages already in the queue?

   > **Answer:** **No.** Existing messages retain the TTL that was applied when they were enqueued (either the per-message TTL or the queue default at enqueue time). The new default TTL only applies to **new messages** sent after the change. This means during a transition, you may have messages with mixed TTL values in the same queue.

3. Why would you disable dead-lettering on expiration for telemetry data?

   > **Answer:** Telemetry data (fuel levels, altitude, speed) is **ephemeral and high-volume**. Once expired, it has no operational value — the current reading supersedes any past reading. Dead-lettering expired telemetry would flood the DLQ with millions of stale readings, wasting storage and making it harder to find genuinely important dead-lettered messages from other queues. Simply discarding expired telemetry is the correct approach.
