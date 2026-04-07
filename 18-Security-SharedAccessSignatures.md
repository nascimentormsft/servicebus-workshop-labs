# Lab 18 – Security: Shared Access Signatures (SAS)

## Objective

Understand **Shared Access Signatures (SAS)** — how they work, how to create scoped SAS policies, and when to use them — demonstrated with a scenario where **third-party ground handling partners** need limited, time-bound access to specific queues.

## Scenario

Transavia contracts ground handling services at different airports. These third-party companies (e.g., Swissport, Menzies, dnata) need to send baggage scan messages to specific queues but should NOT have access to other queues, topics, or management operations. SAS policies provide granular, revocable access tokens.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Namespace-level SAS | **Supported** | **Supported** | **Supported** |
| Entity-level SAS | **Supported** | **Supported** | **Supported** |
| Max SAS rules per entity | 12 | 12 | 12 |

> In production, prefer Managed Identity over SAS. Use SAS only for external partners who cannot use Azure AD, or for short-lived tokens in specific scenarios.

---

## Exercise Steps

### Step 0 — Set Environment Variables

If you haven't already, or if you're starting a new terminal session, set the variables from Lab 01:

```bash
NAMESPACE="sb-transavia-workshop-<your-initials>"
RG="rg-servicebus-workshop"
```

### Step 1 — View the Default Namespace-Level SAS Policy

```bash
az servicebus namespace authorization-rule list \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --output table
```

You'll see `RootManageSharedAccessKey` with Manage + Send + Listen rights.

> **Warning:** The `RootManageSharedAccessKey` is like a master key — it provides full access to everything. NEVER share this with external parties.

### Step 2 — View the Keys for the Default Policy

```bash
az servicebus namespace authorization-rule keys list \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --name RootManageSharedAccessKey \
  --query "{primaryKey: primaryKey, secondaryKey: secondaryKey, primaryConnectionString: primaryConnectionString}" 
```

> Notice there are two keys (primary and secondary). This enables zero-downtime key rotation: update consumers to use the secondary key, then regenerate the primary key.

### Step 3 — Create a Queue-Level SAS Policy for Ground Handling

```bash
# First, ensure the baggage-routing queue exists
az servicebus queue show \
  --name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG 2>/dev/null || \
az servicebus queue create \
  --name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG

# Create a Send-only policy for the ground handler
az servicebus queue authorization-rule create \
  --name ground-handler-send \
  --queue-name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --rights Send
```

### Step 4 — Create a Listen-Only Policy for the Internal Consumer

```bash
az servicebus queue authorization-rule create \
  --name internal-consumer-listen \
  --queue-name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --rights Listen
```

### Step 5 — List All Policies on the Queue

```bash
az servicebus queue authorization-rule list \
  --queue-name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --output table
```

### Step 6 — Get the Connection String for the Ground Handler

```bash
az servicebus queue authorization-rule keys list \
  --name ground-handler-send \
  --queue-name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --query "primaryConnectionString" -o tsv
```

> This connection string only allows sending to the `baggage-routing` queue. The partner cannot receive messages, manage the queue, or access any other queue.

### Step 7 — Test Scoped Access (Portal Verification)

1. Navigate to **Queues** → `baggage-routing`
2. Go to **Shared access policies** (under Settings)
3. You should see both policies:
   - `ground-handler-send` — Send only
   - `internal-consumer-listen` — Listen only
4. Click on each to see the keys and connection strings

### Step 8 — Regenerate a Key (Key Rotation)

```bash
# Regenerate the primary key for the ground handler policy
az servicebus queue authorization-rule keys renew \
  --name ground-handler-send \
  --queue-name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --key PrimaryKey
```

> **Rotation workflow:**
> 1. Share the secondary key with the partner
> 2. Partner updates their config to use secondary key
> 3. Regenerate the primary key
> 4. (Later) share new primary key, regenerate secondary

### Step 9 — Create Multiple Scoped Policies for Different Partners

```bash
# Swissport — Send access to baggage-routing at AMS
az servicebus queue authorization-rule create \
  --name swissport-ams-send \
  --queue-name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --rights Send

# Create a separate queue for another partner
az servicebus queue create \
  --name ground-ops-rtm \
  --namespace-name $NAMESPACE \
  --resource-group $RG

az servicebus queue authorization-rule create \
  --name menzies-rtm-send \
  --queue-name ground-ops-rtm \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --rights Send
```

### Step 10 — Delete a Policy (Revoke Access)

When a ground handling contract ends:

```bash
az servicebus queue authorization-rule delete \
  --name swissport-ams-send \
  --queue-name baggage-routing \
  --namespace-name $NAMESPACE \
  --resource-group $RG
```

> Deleting the policy immediately revokes all access — any connection strings derived from this policy stop working.

---

## SAS Rights Summary

| Right | Allows | Denies |
|-------|--------|--------|
| **Send** | Send messages to the entity | Receive, Browse, Manage |
| **Listen** | Receive and Browse messages | Send, Manage |
| **Manage** | Create/delete/update entities, Send, Listen | Nothing (full access) |

> **Manage** includes Send and Listen implicitly.

---

## SAS vs. Managed Identity

| Aspect | SAS | Managed Identity |
|--------|-----|-----------------|
| Secret management | Keys must be stored and rotated | No secrets |
| Granularity | Entity-level policies | Entity-level RBAC |
| External partner access | **Yes** (shareable tokens) | No (requires Azure AD) |
| Audit | Limited | Full Azure AD audit log |
| Revocation | Delete policy or regenerate keys | Remove role assignment |
| Recommendation | External partners only | All internal applications |

---

## Review Questions

1. An external ground handler needs to send AND receive from a queue. Should you give them a Manage SAS policy?

   > **Answer:** **No.** The `Manage` right includes the ability to create, delete, and modify entities (queues, topics, subscriptions) — far more than the handler needs. Instead, create a SAS policy with **both Send and Listen rights** (but not Manage). This gives the ground handler exactly the access they need without risking them accidentally or maliciously deleting the queue or changing its configuration.

2. What is the maximum number of SAS policies you can have on a single queue?

   > **Answer:** **12.** This applies per entity (queue, topic, or subscription). The namespace level has a separate limit. If you have more than 12 external partners needing individual access, consider: (a) sharing policies between partners with similar access needs, (b) using a proxy/API gateway that authenticates partners and uses a single SAS policy to forward messages, or (c) migrating partners to Azure AD B2B with Managed Identity.

3. Why is key rotation important, and how do dual keys (primary/secondary) help?

   > **Answer:** Key rotation limits the window of exposure if a key is leaked. Dual keys enable **zero-downtime rotation**: (1) share the secondary key with partners, (2) partners update their config to use the secondary key, (3) regenerate the primary key (now invalid, but nobody uses it), (4) the secondary key continues working uninterrupted. Without dual keys, rotation would require simultaneous coordinated updates between key regeneration and all consumer config updates — practically impossible with external partners.
