# Lab 01 – Create Azure Service Bus & Explore Queues

## Objective

Create an Azure Service Bus namespace and set up queues to handle **flight booking confirmation messages** — a core aviation messaging pattern where booking systems need reliable, ordered delivery of passenger reservation data.

## Scenario

Transavia's booking engine processes thousands of reservations daily. Each booking confirmation must be reliably delivered to downstream systems (e-ticket generation, payment processing, loyalty points). A Service Bus Queue guarantees each message is processed exactly once.

---

## Tier Considerations

| Tier | Max Message Size | Max Namespace Size | Queues | Price |
|------|------------------|--------------------|--------|-------|
| **Basic** | 256 KB | — | 1,000 | Lowest |
| **Standard** | 256 KB | 1–80 GB | 10,000 | Mid |
| **Premium** | 1 MB | 1–4 TB | 10,000 | Highest |

**For this lab:** Standard tier is sufficient. In production, Premium would be chosen for mission-critical booking flows due to dedicated resources and predictable latency.

> **Key consideration:** Basic tier does NOT support Topics/Subscriptions, only Queues. If you anticipate pub/sub scenarios (e.g., flight status updates to multiple consumers), start with Standard or Premium.

---

## Prerequisites

- Azure subscription with Contributor access
- Azure CLI installed or access to Azure Cloud Shell

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 00:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create a Queue for Booking Confirmations

```bash
az servicebus queue create \
  --name booking-confirmations \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --max-size 1024 \
  --default-message-time-to-live P14D
```

> The `--default-message-time-to-live P14D` sets the default TTL to 14 days (ISO 8601 duration).

### Step 2 — Create a Second Queue for Baggage Processing

```bash
az servicebus queue create \
  --name baggage-processing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --max-size 2048 \
  --enable-dead-lettering-on-message-expiration true
```

### Step 3 — List All Queues

```bash
az servicebus queue list \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --output table
```

### Step 4 — Inspect Queue Properties

```bash
az servicebus queue show \
  --name booking-confirmations \
  --namespace-name $NAMESPACE \
  --resource-group $RG
```

Review the JSON output and identify:
- `maxSizeInMegabytes`
- `defaultMessageTimeToLive`
- `lockDuration`
- `deadLetteringOnMessageExpiration`

### Step 5 — Send a Test Message via Service Bus Explorer

1. In the Azure Portal, navigate to your Service Bus namespace
2. Click on **Queues** → `booking-confirmations`
3. Click **Service Bus Explorer** in the left menu
4. Click **Send messages**
5. Set **Content Type** to `application/json`
6. Paste the following message body:

```json
{
  "bookingId": "TV-2026-040701",
  "passengerName": "Jan de Vries",
  "flightNumber": "HV6321",
  "route": "AMS-BCN",
  "departureDate": "2026-05-15",
  "class": "Economy",
  "status": "Confirmed"
}
```

7. Click **Send**

### Step 6 — Peek at the Message

1. In Service Bus Explorer, switch to **Peek** mode
2. Click **Peek from start**
3. Verify the message content matches what you sent
4. Note that peeking does NOT remove the message from the queue

### Step 7 — Receive the Message

1. Switch to **Receive** mode
2. Select **ReceiveAndDelete** mode
3. Click **Receive**
4. Confirm the message is received and the queue count drops to 0

---

## Cleanup

> Keep resources for the next labs. We will clean up at the end of the workshop.

---

## Review Questions

1. What happens if you try to send a message larger than 256 KB on a Standard tier namespace?

   > **Answer:** The send operation is rejected with a `MessageSizeExceededException`. Standard and Basic tiers have a 256 KB maximum message size. To send larger messages, you need to upgrade to Premium tier (up to 100 MB) or implement a claim-check pattern where the payload is stored in Azure Blob Storage and only a reference is sent via Service Bus.

2. Why would you enable dead-lettering on message expiration for the baggage-processing queue?

   > **Answer:** Expired baggage messages may indicate a processing failure or system outage. By dead-lettering them instead of silently discarding, the operations team can investigate why messages weren't processed in time, identify the root cause, and potentially resubmit them. This is critical in aviation where lost baggage information directly impacts passengers.

3. What is the difference between **Peek** and **Receive** in Service Bus Explorer?

   > **Answer:** **Peek** reads the message without locking or removing it — it's non-destructive and safe for troubleshooting in production. **Receive** either locks the message (PeekLock mode) or permanently removes it (ReceiveAndDelete mode). Peek does not affect the message count or state; Receive does.
