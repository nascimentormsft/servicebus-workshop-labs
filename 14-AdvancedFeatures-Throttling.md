# Lab 14 – Throttling

## Objective

Understand Service Bus **throttling behavior** and how to handle it — demonstrated with a **flight disruption scenario** where a major weather event causes a spike in messages across all systems simultaneously.

## Scenario

A major storm system hits Amsterdam, causing 50+ flight cancellations and delays in a 30-minute window. Every system (passenger notifications, crew rescheduling, baggage rerouting, refund processing) floods Service Bus with messages simultaneously. Understanding throttling — when Service Bus pushes back — is critical for designing resilient aviation systems.

---

## Tier Considerations

| Feature | Standard | Premium |
|---------|----------|---------|
| Throughput | Shared, variable | Dedicated, predictable |
| Throttling | **Yes** (HTTP 429 / ServerBusy) | **Rare** (dedicated resources) |
| Messaging Units | N/A | 1, 2, 4, 8, 16 MUs |
| Max throughput per MU | N/A | ~1,000 msg/sec (1 KB) |

> **This is the #1 reason to choose Premium for aviation.** Standard tier shares resources with other tenants. During industry-wide disruptions, your namespace competes for resources. Premium provides dedicated compute and memory.

---

## Exercise Steps

### Step 1 — Check Your Current Namespace SKU

```bash
az servicebus namespace show \
  --name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --query "{sku: sku.name, tier: sku.tier, capacity: sku.capacity}"
```

### Step 2 — Understand Throttling Triggers

Service Bus throttles when:

| Trigger | Standard Tier | Premium Tier |
|---------|--------------|--------------|
| CPU usage > 70% | N/A (shared) | Yes — scale up MUs |
| Memory usage > 70% | N/A (shared) | Yes — scale up MUs |
| Namespace storage > limit | Yes | Yes |
| Request rate exceeds capacity | Yes (variable limit) | Yes (higher, predictable limit) |

### Step 3 — View Throttling Metrics in Azure Monitor

1. Navigate to your Service Bus namespace in the Portal
2. Go to **Monitoring** → **Metrics**
3. Add the following metrics:
   - **Throttled Requests** (Count)
   - **Server Errors** (Count)
   - **Incoming Requests** (Count)
4. Set the time range to **Last hour**
5. This baseline should show zero throttled requests

### Step 4 — Simulate Load (Optional, Standard Tier)

> **Warning:** Only do this on a workshop namespace, never on production.

If you have Azure Cloud Shell and want to see throttling in action:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"

# Get connection string
CONN=$(az servicebus namespace authorization-rule keys list \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv)

echo "Use this connection string with a load testing tool to simulate high message volumes."
echo "Connection: $CONN"
```

### Step 5 — Understand the Throttling Response

When throttled, Service Bus returns:

- **AMQP**: `amqp:resource-limit-exceeded` with a `Retry-After` header
- **REST/HTTP**: HTTP `429 Too Many Requests` with `Retry-After` header
- **SDK exception**: `ServiceBusException` with `Reason = ServiceBusy`

### Step 6 — Review Retry Policies

In the Portal, you can't configure retry policies, but in code:

```csharp
// SDK built-in retry handles throttling automatically
var clientOptions = new ServiceBusClientOptions
{
    RetryOptions = new ServiceBusRetryOptions
    {
        Mode = ServiceBusRetryMode.Exponential,
        MaxRetries = 10,
        Delay = TimeSpan.FromSeconds(0.8),
        MaxDelay = TimeSpan.FromMinutes(1),
        TryTimeout = TimeSpan.FromSeconds(60)
    }
};
```

### Step 7 — Premium Tier: Scaling Messaging Units

If you have a Premium namespace (or viewing documentation):

```bash
# View current capacity (Premium only)
az servicebus namespace show \
  --name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --query "sku.capacity"

# Scale up Premium namespace (Premium only)
# az servicebus namespace update \
#   --name sb-transavia-premium \
#   --resource-group rg-servicebus-workshop \
#   --capacity 4
```

Navigate in Portal: **Settings** → **Scale** to see the Messaging Units slider (Premium only).

### Step 8 — Design Patterns to Handle Throttling

Review these patterns commonly used in aviation systems:

```
Pattern 1: Circuit Breaker
├── Normal: Send messages normally
├── Open (throttled): Stop sending, queue locally
└── Half-open: Test with single message, resume if OK

Pattern 2: Load Leveling
├── Burst traffic → intermediate queue → steady consumer
└── The queue itself IS the load leveler

Pattern 3: Priority Queues
├── safety-critical queue (Premium, high MU)
├── operational queue (Premium, standard MU)
└── informational queue (Standard tier, throttling OK)
```

### Step 9 — Set Up a Throttling Alert

1. In the Portal, navigate to namespace → **Monitoring** → **Alerts**
2. Click **+ Create alert rule**
3. Condition: **Throttled Requests** > 0
4. Action group: email to ops team
5. Severity: **Warning (Sev 2)**

```bash
# Or via CLI:
az monitor metrics alert create \
  --name "servicebus-throttling-alert" \
  --resource-group rg-servicebus-workshop \
  --scopes "/subscriptions/<sub-id>/resourceGroups/rg-servicebus-workshop/providers/Microsoft.ServiceBus/namespaces/sb-transavia-workshop-<your-initials>" \
  --condition "total ThrottledRequests > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --description "Alert when Service Bus requests are being throttled"
```

---

## Premium Tier Sizing for Aviation

| Scenario | Estimated Load | Recommended MUs |
|----------|---------------|-----------------|
| Normal operations (1 airport) | 100-500 msg/sec | 1 MU |
| Normal operations (multi-airport) | 500-2,000 msg/sec | 2 MUs |
| Disruption (weather event) | 2,000-5,000 msg/sec | 4 MUs |
| Major disruption + recovery | 5,000-10,000 msg/sec | 8 MUs |
| Peak (holiday + disruption) | 10,000+ msg/sec | 16 MUs |

> **Auto-scale:** Premium doesn't auto-scale. Set up alerts on CPU/memory metrics and define a runbook to scale MUs when thresholds are crossed.

---

## Review Questions

1. Why is Standard tier throttling unpredictable compared to Premium?
2. During a storm causing 50 flight cancellations, which messages should be prioritized? How would you design this?
3. What is the cost trade-off of running 4 MUs continuously vs. 1 MU with manual scaling during disruptions?
