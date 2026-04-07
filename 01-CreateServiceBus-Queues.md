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

### Step 1 — Create a Resource Group

```bash
az group create \
  --name rg-servicebus-workshop \
  --location westeurope
```

### Step 2 — Create a Service Bus Namespace

```bash
az servicebus namespace create \
  --name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --sku Standard \
  --location westeurope
```

> **Note:** Namespace names must be globally unique. Append your initials or a random suffix.

### Step 3 — Verify the Namespace in the Portal

1. Open the [Azure Portal](https://portal.azure.com)
2. Navigate to **Resource Groups** → `rg-servicebus-workshop`
3. Click on the Service Bus namespace
4. Observe the **Overview** blade: note the tier, location, and FQDN

### Step 4 — Create a Queue for Booking Confirmations

```bash
az servicebus queue create \
  --name booking-confirmations \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --max-size 1024 \
  --default-message-time-to-live P14D
```

> The `--default-message-time-to-live P14D` sets the default TTL to 14 days (ISO 8601 duration).

### Step 5 — Create a Second Queue for Baggage Processing

```bash
az servicebus queue create \
  --name baggage-processing \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --max-size 2048 \
  --enable-dead-lettering-on-message-expiration true
```

### Step 6 — List All Queues

```bash
az servicebus queue list \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --output table
```

### Step 7 — Inspect Queue Properties

```bash
az servicebus queue show \
  --name booking-confirmations \
  --namespace-name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop
```

Review the JSON output and identify:
- `maxSizeInMegabytes`
- `defaultMessageTimeToLive`
- `lockDuration`
- `deadLetteringOnMessageExpiration`

### Step 8 — Send a Test Message via Service Bus Explorer

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

### Step 9 — Peek at the Message

1. In Service Bus Explorer, switch to **Peek** mode
2. Click **Peek from start**
3. Verify the message content matches what you sent
4. Note that peeking does NOT remove the message from the queue

### Step 10 — Receive the Message

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
2. Why would you enable dead-lettering on message expiration for the baggage-processing queue?
3. What is the difference between **Peek** and **Receive** in Service Bus Explorer?
