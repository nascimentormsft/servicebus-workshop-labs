# Lab 15 – Partitioning

## Objective

Understand Service Bus **partitioning** for improved availability and throughput — demonstrated with a **high-volume baggage tracking system** that processes millions of scan events per day across multiple airports.

## Scenario

Transavia's baggage tracking system processes barcode scans at every touchpoint: check-in counter, belt, cart, aircraft hold, arrival belt. During peak summer operations, this generates massive message volumes. Partitioning spreads messages across multiple message brokers and stores, increasing throughput and availability.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Partitioned Queues/Topics | Not supported | **Supported** | **Supported** |
| Number of Partitions | — | 16 (fixed) | Varies by MU |
| Max Entity Size (partitioned) | — | 80 GB (16 × 5 GB) | Up to 80 GB |

> **Important:** As of current Azure updates, Premium tier supports partitioning through its dedicated resources (Messaging Units). The approach differs from Standard tier's 16-partition model. Check the latest Azure documentation for Premium partitioning specifics.

> **Key decision:** Partitioning CANNOT be changed after entity creation. A partitioned queue cannot be converted to non-partitioned, and vice versa.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create a Partitioned Queue

```bash
az servicebus queue create \
  --name baggage-tracking-partitioned \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --enable-partitioning true \
  --max-size 4096
```

### Step 2 — Create a Non-Partitioned Queue for Comparison

```bash
az servicebus queue create \
  --name baggage-tracking-standard \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --enable-partitioning false \
  --max-size 1024
```

### Step 3 — Compare Properties

```bash
# Partitioned queue
az servicebus queue show \
  --name baggage-tracking-partitioned \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "{name: name, partitioning: enablePartitioning, maxSize: maxSizeInMegabytes}"

# Non-partitioned queue
az servicebus queue show \
  --name baggage-tracking-standard \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "{name: name, partitioning: enablePartitioning, maxSize: maxSizeInMegabytes}"
```

> Notice: The partitioned queue's `maxSizeInMegabytes` is the total across all 16 partitions, not per partition.

### Step 4 — Understand Partitioning Architecture

```
Non-Partitioned Queue:
┌─────────────────────────────────────┐
│           Single Message Store       │
│  [msg1] [msg2] [msg3] [msg4] [msg5] │
└─────────────────────────────────────┘
  Single point of failure, single throughput path

Partitioned Queue (16 partitions):
┌────────────┐ ┌────────────┐ ┌────────────┐     ┌────────────┐
│ Partition 0 │ │ Partition 1 │ │ Partition 2 │ ... │Partition 15│
│  [msg1]     │ │  [msg2]     │ │  [msg3]     │     │  [msg16]   │
│  [msg17]    │ │  [msg18]    │ │  [msg19]    │     │  [msg32]   │
└────────────┘ └────────────┘ └────────────┘     └────────────┘
  16 independent stores = higher throughput & availability
```

### Step 5 — Send Messages and Observe Partition Distribution

Using Service Bus Explorer, send to `baggage-tracking-partitioned`:

**Message 1 — No partition key (round-robin):**
```json
{
  "scanId": "SCAN-001",
  "baggageTag": "TV0001001",
  "flightNumber": "HV6321",
  "scanPoint": "CheckIn",
  "airport": "AMS",
  "timestamp": "2026-04-07T12:00:00Z"
}
```

**Message 2 — With partition key:**
- Set custom property `PartitionKey` = `HV6321`
```json
{
  "scanId": "SCAN-002",
  "baggageTag": "TV0001002",
  "flightNumber": "HV6321",
  "scanPoint": "Belt",
  "airport": "AMS",
  "timestamp": "2026-04-07T12:15:00Z"
}
```

**Message 3 — Different partition key:**
- Set custom property `PartitionKey` = `HV5102`
```json
{
  "scanId": "SCAN-003",
  "baggageTag": "TV0002001",
  "flightNumber": "HV5102",
  "scanPoint": "CheckIn",
  "airport": "RTM",
  "timestamp": "2026-04-07T12:30:00Z"
}
```

### Step 6 — Understand Partition Key Behavior

| Partition Key Set? | Behavior |
|---|---|
| **No** | Message assigned to partition via round-robin |
| **Yes** | All messages with same partition key go to same partition |
| **Session ID set** | Session ID used as partition key (correlated) |

> **Aviation use case:** Use `flightNumber` as partition key so all baggage scans for the same flight land in the same partition — enabling efficient bulk queries for flight-level reconciliation.

### Step 7 — Partitioned Topics

```bash
az servicebus topic create \
  --name baggage-events-partitioned \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --enable-partitioning true
```

### Step 8 — Inspect in Portal

1. Navigate to **Queues** → `baggage-tracking-partitioned`
2. Look at Properties — note **Enable Partitioning: true**
3. Compare the **Max size** with the non-partitioned version

---

## Partitioning Trade-offs

| Aspect | Partitioned | Non-Partitioned |
|--------|------------|-----------------|
| **Throughput** | Higher (16× paths) | Single path |
| **Availability** | Higher (survives partition failures) | Lower |
| **Ordering** | Guaranteed within partition only | Guaranteed (FIFO) |
| **Transactions** | Limited (within single partition) | Full support |
| **Message size** | Max 1 MB (Standard) | Max 256 KB/100 MB |
| **Sessions** | Supported (session ID = partition key) | Supported |
| **Duplicate detection** | Per partition | Global |

> **Critical for aviation:** If you need strict FIFO ordering across ALL messages (not just within a partition), do NOT use partitioning. Use sessions on a non-partitioned queue instead.

---

## When to Partition in Aviation

| Use Case | Partition? | Reasoning |
|----------|----------|-----------|
| Baggage tracking scans | **Yes** | High volume, partition by flight |
| Passenger notifications | **Yes** | High volume during disruptions |
| Flight plan filing | **No** | Low volume, needs strict ordering |
| Crew assignments | **No** | Needs transactions, sessions |
| Payment processing | **No** | Needs transactions, global dedup |

---

## Review Questions

1. A partitioned queue has duplicate detection enabled. A message with the same MessageId is sent to two different partitions. Will duplicate detection catch it?
2. Why is the maximum entity size larger for partitioned queues?
3. Can you convert a non-partitioned queue to partitioned without data loss?
