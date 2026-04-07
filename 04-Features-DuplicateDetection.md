# Lab 04 – Duplicate Detection

## Objective

Enable and test **duplicate detection** on a Service Bus queue to prevent double-processing of **payment transactions** — a critical requirement in aviation where duplicate charges to passengers must be avoided.

## Scenario

The payment gateway for Transavia's booking system occasionally retries transactions due to network timeouts. Without duplicate detection, a passenger could be charged twice for the same booking. Service Bus can automatically discard duplicate messages based on the `MessageId`, within a configurable time window.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Duplicate Detection | Not supported | **Supported** | **Supported** |
| Max Detection History Window | — | 7 days | 7 days |

> For payment processing in production, Premium tier is recommended due to its dedicated resources and network isolation. The duplicate detection window should match your retry policy timeout.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Create a Queue with Duplicate Detection Enabled

```bash
az servicebus queue create \
  --name payment-transactions \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --enable-duplicate-detection true \
  --duplicate-detection-history-time-window P1D
```

> `--duplicate-detection-history-time-window P1D` sets a 1-day window. Duplicates are only caught within this window.

> **Note:** Like sessions, duplicate detection CANNOT be toggled after queue creation. Plan ahead.

### Step 2 — Verify in the Portal

1. Navigate to **Queues** → `payment-transactions`
2. In the **Properties** section, confirm:
   - **Requires duplicate detection**: `true`
   - **Duplicate detection history time window**: `1 day`

### Step 3 — Send the First Payment Message

Using Service Bus Explorer:

1. Go to **Queues** → `payment-transactions` → **Service Bus Explorer**
2. Click **Send messages**
3. Set the **Message ID** to: `PAY-2026-040701-HV6321-JDV`
4. Set **Content Type** to `application/json`
5. Paste:

```json
{
  "transactionId": "PAY-2026-040701-HV6321-JDV",
  "bookingRef": "TV-2026-040701",
  "passengerName": "Jan de Vries",
  "amount": 189.50,
  "currency": "EUR",
  "flightNumber": "HV6321",
  "paymentMethod": "CreditCard",
  "cardLast4": "4532"
}
```

6. Click **Send**
7. Verify the queue shows **1 active message**

### Step 4 — Send a Duplicate (Same Message ID)

1. Send the **exact same message** again with the **same Message ID**: `PAY-2026-040701-HV6321-JDV`
2. Click **Send**
3. Note that the send **succeeds without error** (the broker silently discards the duplicate)
4. Check the queue — it should still show **1 active message**, not 2

### Step 5 — Send a Different Payment (Different Message ID)

1. Send a new message with **Message ID**: `PAY-2026-040701-HV5102-AK`
2. Body:

```json
{
  "transactionId": "PAY-2026-040701-HV5102-AK",
  "bookingRef": "TV-2026-040702",
  "passengerName": "Anna Kuijpers",
  "amount": 245.00,
  "currency": "EUR",
  "flightNumber": "HV5102",
  "paymentMethod": "iDEAL"
}
```

3. The queue should now show **2 active messages**

### Step 6 — Verify via CLI

```bash
az servicebus queue show \
  --name payment-transactions \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "{activeMessageCount: countDetails.activeMessageCount, duplicateDetection: requiresDuplicateDetection, duplicateWindow: duplicateDetectionHistoryTimeWindow}"
```

### Step 7 — Test with Azure CLI Message Count

```bash
az servicebus queue show \
  --name payment-transactions \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "countDetails.activeMessageCount" \
  --output tsv
```

This should return `2`, confirming the duplicate was discarded.

---

## How Duplicate Detection Works

1. Service Bus maintains a hash table of all `MessageId` values received within the detection window
2. When a new message arrives, its `MessageId` is checked against this table
3. If found → message is silently discarded (accepted but not enqueued)
4. If not found → message is enqueued and `MessageId` is added to the table
5. Entries older than the detection window are automatically removed

---

## Key Takeaways

- Duplicate detection is based on the **MessageId** property
- The sender must set a **consistent, deterministic MessageId** (e.g., transaction ID)
- Duplicates are silently discarded — no error is returned to the sender
- The detection window is configurable (up to 7 days) but affects memory usage
- This is a **broker-side** feature — no consumer-side logic needed

---

## Review Questions

1. What would happen if the sender uses `Guid.NewGuid()` as the MessageId for retries?
2. The duplicate detection window is set to 1 day. A network issue causes a retry after 25 hours. Will the duplicate be caught?
3. Does duplicate detection add latency to message processing? What is the trade-off?
