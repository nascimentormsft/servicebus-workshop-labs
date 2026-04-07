# Lab 17 – Security: Managed Identity

## Objective

Configure **Managed Identity** authentication for Service Bus, eliminating connection strings and SAS keys — demonstrated with an **Azure Function processing flight booking messages** that authenticates to Service Bus using its system-assigned identity.

## Scenario

Hardcoded connection strings and SAS keys in application code are a major security risk — especially in aviation where compromised credentials could allow unauthorized access to booking, passenger, or operational data. Managed Identity provides automatic, secure authentication without managing secrets.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Managed Identity authentication | **Supported** | **Supported** | **Supported** |
| RBAC roles | **Supported** | **Supported** | **Supported** |
| Disable SAS keys (local auth) | **Supported** | **Supported** | **Supported** |

> Managed Identity works on all tiers. In production, combine with `disableLocalAuth = true` to ensure ONLY Managed Identity/Azure AD is used (no SAS keys).

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — Understand Authentication Methods

```
Method 1: Connection String (SAS Key) ❌ Not Recommended
├── Secret stored in config/code
├── Risk: leaked key = full access
├── No audit trail of WHO accessed
└── Key rotation is manual and disruptive

Method 2: Managed Identity ✅ Recommended
├── No secrets to manage
├── Identity is tied to the Azure resource
├── Full audit trail via Azure AD logs
└── Automatic credential rotation
```

### Step 2 — View Current Authorization Rules (SAS Keys)

```bash
# List all SAS authorization rules
az servicebus namespace authorization-rule list \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --output table
```

You'll see the default `RootManageSharedAccessKey` with Manage, Send, and Listen rights.

### Step 3 — Review Service Bus RBAC Roles

```bash
# List available Service Bus roles
az role definition list \
  --query "[?contains(roleName, 'Service Bus')].{roleName: roleName, description: description}" \
  --output table
```

Key roles:

| Role | Permissions | Use Case |
|------|-------------|----------|
| **Azure Service Bus Data Owner** | Send + Receive + Manage | Admin, deployment pipelines |
| **Azure Service Bus Data Sender** | Send only | Producer applications |
| **Azure Service Bus Data Receiver** | Receive only | Consumer applications |

### Step 4 — Create a Test Resource with Managed Identity

We'll create a simple Azure Function app concept (simulated via CLI):

```bash
# Create a user-assigned managed identity (simulating an app identity)
az identity create \
  --name mi-booking-processor \
  --resource-group $RG \
  --location westeurope
```

### Step 5 — Assign the Receiver Role to the Managed Identity

```bash
# Get the identity's principal ID
PRINCIPAL_ID=$(az identity show \
  --name mi-booking-processor \
  --resource-group $RG \
  --query principalId -o tsv)

# Get the Service Bus namespace resource ID
SB_ID=$(az servicebus namespace show \
  --name $NAMESPACE \
  --resource-group $RG \
  --query id -o tsv)

# Assign "Azure Service Bus Data Receiver" role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Azure Service Bus Data Receiver" \
  --scope $SB_ID
```

### Step 6 — Assign Sender Role for a Different Identity

```bash
# Create a sender identity (simulating the booking engine)
az identity create \
  --name mi-booking-engine \
  --resource-group $RG \
  --location westeurope

SENDER_PRINCIPAL=$(az identity show \
  --name mi-booking-engine \
  --resource-group $RG \
  --query principalId -o tsv)

# Assign "Azure Service Bus Data Sender" role
az role assignment create \
  --assignee $SENDER_PRINCIPAL \
  --role "Azure Service Bus Data Sender" \
  --scope $SB_ID
```

### Step 7 — Verify Role Assignments

```bash
az role assignment list \
  --scope $SB_ID \
  --query "[].{principal: principalName, role: roleDefinitionName}" \
  --output table
```

### Step 8 — Scope Roles to Specific Queues/Topics

For least-privilege, scope roles to individual queues instead of the namespace:

```bash
# Get queue resource ID
QUEUE_ID=$(az servicebus queue show \
  --name booking-confirmations \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query id -o tsv)

# Assign receiver role only for booking-confirmations queue
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Azure Service Bus Data Receiver" \
  --scope $QUEUE_ID
```

### Step 9 — Verify in the Portal: Access Control (IAM)

1. Navigate to your Service Bus namespace
2. Click **Access control (IAM)**
3. Click **Role assignments** tab
4. You should see the role assignments you created
5. Click on each to see the scope (namespace vs. specific queue)

### Step 10 — Review How Applications Use Managed Identity

In code, there are NO connection strings:

```csharp
// .NET — Managed Identity authentication
using Azure.Identity;
using Azure.Messaging.ServiceBus;

// No connection string needed!
var client = new ServiceBusClient(
    "sb-transavia-workshop.servicebus.windows.net",  // Just the hostname
    new DefaultAzureCredential()  // Uses Managed Identity automatically
);

var sender = client.CreateSender("booking-confirmations");
```

```python
# Python — Managed Identity authentication
from azure.identity import DefaultAzureCredential
from azure.servicebus import ServiceBusClient

credential = DefaultAzureCredential()
client = ServiceBusClient(
    fully_qualified_namespace="sb-transavia-workshop.servicebus.windows.net",
    credential=credential
)
```

---

## Least-Privilege Role Assignment Strategy

| System | Role | Scope |
|--------|------|-------|
| Booking engine | Sender | `booking-confirmations` queue |
| Booking processor | Receiver | `booking-confirmations` queue |
| Notification service | Receiver | `passenger-notifications` subscription |
| Ops dashboard | Receiver | `flight-status-updates` topic (all subs) |
| Deployment pipeline | Owner | Namespace (for management) |
| Monitoring system | Reader | Namespace (metrics only) |

---

## Key Takeaways

- Managed Identity eliminates secrets from code and configuration
- Use RBAC to assign the minimum required permissions (Sender, Receiver, or Owner)
- Scope roles to the most granular level possible (queue > topic > namespace)
- In production, disable local auth (SAS keys) after migrating to Managed Identity
- Role assignments take up to 10 minutes to propagate

---

## Review Questions

1. An application needs to both send and receive from the same queue. Which role(s) should you assign?
2. What happens if you disable local auth and an application still uses a connection string?
3. Why is queue-level scoping more secure than namespace-level?
