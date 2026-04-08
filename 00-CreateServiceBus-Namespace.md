# Lab 00 – Create Azure Service Bus Namespace

## Objective

Create the **Resource Group** and **Service Bus Namespace** that will be used throughout all workshop labs, and assign the necessary **RBAC roles** so you can use Service Bus Explorer in the Azure Portal.

## Scenario

Before diving into queues, topics, and advanced features, every participant needs a working Service Bus namespace. This lab sets up the shared foundation: a resource group, a Standard-tier namespace with local authentication enabled, and the role assignments required to send and receive messages through the Portal.

---

## Tier Considerations

| Tier | Max Message Size | Max Namespace Size | Topics/Subscriptions | Price |
|------|------------------|--------------------|----------------------|-------|
| **Basic** | 256 KB | — | Not supported | Lowest |
| **Standard** | 256 KB | 1–80 GB | Supported | Mid |
| **Premium** | 100 MB | 1–4 TB | Supported | Highest |

**For this workshop:** Standard tier gives us access to all features (queues, topics, sessions, filters, auto-forwarding) at a reasonable cost.

> **Key consideration:** Basic tier does NOT support Topics/Subscriptions. If you anticipate pub/sub scenarios (e.g., flight status updates to multiple consumers), start with Standard or Premium.

---

## Prerequisites

- Azure subscription with Contributor access
- Azure CLI installed or access to Azure Cloud Shell

---

## Exercise Steps

### Step 1 — Set Environment Variables

Run this once at the start of the workshop. These variables will be used throughout **all labs**.

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

> **Note:** Replace `<your-initials>` with your actual initials (e.g., `sb-transavia-workshop-rn`). Namespace names must be globally unique.

### Step 2 — Create a Resource Group

```bash
az group create \
  --name $RG \
  --location westeurope
```

### Step 3 — Create a Service Bus Namespace

```bash
az servicebus namespace create \
  --name $NAMESPACE \
  --resource-group $RG \
  --sku Standard \
  --location westeurope \
  --disable-local-auth false
```

> **Why `--disable-local-auth false`?** This explicitly enables SAS key (local) authentication, which is required for **Service Bus Explorer** in the Azure Portal. Without it, you won't be able to send/receive/peek messages through the portal.

### Step 4 — Verify the Namespace in the Portal

1. Open the [Azure Portal](https://portal.azure.com)
2. Navigate to **Resource Groups** → your resource group
3. Click on the Service Bus namespace
4. Observe the **Overview** blade: note the tier, location, and FQDN (`sb-transavia-workshop-<your-initials>.servicebus.windows.net`)

### Step 5 — Assign RBAC Roles for Service Bus Explorer

To use Service Bus Explorer in the Portal for sending and receiving messages, your user account needs the following roles:

```bash
# Get your Azure AD user object ID
USER_ID=$(az ad signed-in-user show --query id -o tsv)

# Get the Service Bus namespace resource ID
SB_ID=$(az servicebus namespace show \
  --name $NAMESPACE \
  --resource-group $RG \
  --query id -o tsv)

# Assign "Azure Service Bus Data Sender" role
az role assignment create \
  --assignee $USER_ID \
  --role "Azure Service Bus Data Sender" \
  --scope $SB_ID

# Assign "Azure Service Bus Data Receiver" role
az role assignment create \
  --assignee $USER_ID \
  --role "Azure Service Bus Data Receiver" \
  --scope $SB_ID
```

> **Why two roles?** Azure Service Bus uses separate roles for sending and receiving. This follows the principle of least privilege — a producer application would only get the Sender role, a consumer only the Receiver role. For this workshop, we need both.

### Step 6 — Verify Role Assignments

```bash
az role assignment list \
  --assignee $USER_ID \
  --scope $SB_ID \
  --output table
```

You should see two entries:
- `Azure Service Bus Data Sender`
- `Azure Service Bus Data Receiver`

You can also verify in the Portal:
1. Navigate to your Service Bus namespace
2. Go to **Access control (IAM)** → **Role assignments**
3. Find your user with both roles listed

### Step 7 — Test Service Bus Explorer Access

1. Navigate to your Service Bus namespace in the Portal
2. Click **Service Bus Explorer** in the left menu
3. You should see the explorer interface without authentication errors

> **⚠ Note:** Role assignments can take **up to 5 minutes** to propagate. If you get a permissions error in Service Bus Explorer, wait a few minutes and refresh the page.

---

## Available RBAC Roles for Service Bus

| Role | Permissions | Typical Use |
|------|-------------|-------------|
| **Azure Service Bus Data Sender** | Send messages to queues/topics | Producer applications |
| **Azure Service Bus Data Receiver** | Receive/peek messages from queues/subscriptions | Consumer applications |
| **Azure Service Bus Data Owner** | Full data-plane access (send, receive, manage) | Admin, CI/CD pipelines |

> **Best practice for production:** Assign roles at the **entity level** (specific queue/topic) rather than namespace level for tighter security. For this workshop, namespace-level is fine.

---

## Key Takeaways

- The namespace is the top-level container for all Service Bus entities (queues, topics, subscriptions)
- Standard tier supports all features needed for this workshop
- Local authentication (SAS) must be enabled for Portal Service Bus Explorer
- RBAC roles (Data Sender + Data Receiver) are required to send and receive messages through the Portal
- Role assignments can take a few minutes to propagate

---

## Review Questions

1. Why do we need both local authentication (SAS) AND RBAC role assignments?

   > **Answer:** These are two separate authentication/authorization mechanisms. **Local authentication (SAS keys)** enables the Portal's Service Bus Explorer to connect to the namespace at the transport level. **RBAC roles** authorize your Azure AD identity to perform specific data-plane operations (send, receive). Both must be in place: SAS for the connection, RBAC for the permission to act on messages. In a production environment, you would typically disable local auth and use only RBAC with Managed Identities.

2. What is the difference between "Azure Service Bus Data Owner" and assigning both Sender and Receiver roles?

   > **Answer:** **Data Owner** includes send, receive, AND manage permissions (create/delete queues, update rules, etc.). Sender + Receiver only allows message operations, NOT entity management. Data Owner is overly permissive for most scenarios — it violates least privilege. Use it only for admin tooling or CI/CD pipelines that need to create infrastructure.

3. What happens if you try to use Service Bus Explorer without the Data Receiver role?

   > **Answer:** You can still navigate to the Service Bus Explorer UI, but **Peek** and **Receive** operations will fail with an `Unauthorized` error. You would be able to **Send** messages (if you have the Sender role) but not read them back. This is a common missed step that causes confusion during workshops.
