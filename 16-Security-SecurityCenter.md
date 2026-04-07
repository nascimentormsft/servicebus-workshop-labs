# Lab 16 – Security: Microsoft Defender for Cloud (Security Center)

## Objective

Enable and explore **Microsoft Defender for Cloud** (formerly Azure Security Center) recommendations for Service Bus — demonstrated with a security audit of Transavia's messaging infrastructure to ensure it meets aviation security standards.

## Scenario

Aviation systems are high-value targets for cyber attacks. EASA (European Union Aviation Safety Agency) and national CAAs require airlines to maintain robust cybersecurity. Microsoft Defender for Cloud provides continuous security assessment and recommendations for Service Bus namespaces, helping identify misconfigurations and vulnerabilities.

---

## Tier Considerations

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Defender for Cloud Support | **Yes** | **Yes** | **Yes** |
| Private endpoints | No | No | **Yes** |
| Virtual Network integration | No | No | **Yes** |
| IP filtering | No | **Partial** | **Yes** |

> **For aviation:** Premium tier is strongly recommended for security-sensitive workloads because it supports Private Endpoints (no public internet exposure), VNet integration, and customer-managed encryption keys.

---

## Exercise Steps

### Step 1 — View Security Recommendations for Service Bus

1. In the Azure Portal, navigate to **Microsoft Defender for Cloud**
2. Click **Recommendations** in the left menu
3. Filter by resource type: **Service Bus**
4. Review the list of recommendations for your namespace

Common recommendations you may see:

| Recommendation | Severity | Description |
|---|---|---|
| Service Bus namespaces should use private link | High | Public endpoints are accessible from internet |
| Service Bus should use CMK encryption | Medium | Data at rest should use customer-managed keys |
| Diagnostic logs should be enabled | Medium | Needed for security investigation |
| Service Bus should restrict network access | High | Minimize attack surface |

### Step 2 — Check the Security Posture of Your Namespace

```bash
az servicebus namespace show \
  --name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --query "{name: name, sku: sku.name, publicNetworkAccess: publicNetworkAccess, minimumTlsVersion: minimumTlsVersion, disableLocalAuth: disableLocalAuth}"
```

### Step 3 — Review Network Access Settings

1. Navigate to your Service Bus namespace in the Portal
2. Go to **Networking** (under Settings)
3. Review the current configuration:
   - **Public network access**: Probably "Enabled" (workshop default)
   - **Selected networks**: Any IP filter rules?
   - **Private endpoint connections**: None (Standard tier doesn't support these)

### Step 4 — Enable Minimum TLS Version

Ensure the namespace requires TLS 1.2 (minimum for aviation security compliance):

```bash
az servicebus namespace update \
  --name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --minimum-tls-version 1.2
```

### Step 5 — Disable Local Authentication (SAS Keys)

For higher security, disable SAS key authentication and force Microsoft Entra ID (Azure AD):

```bash
# Check current state
az servicebus namespace show \
  --name sb-transavia-workshop-<your-initials> \
  --resource-group rg-servicebus-workshop \
  --query "disableLocalAuth"

# NOTE: Only do this if all consumers use Azure AD. 
# For workshop purposes, we'll leave it enabled.
# In production: az servicebus namespace update --disable-local-auth true
```

### Step 6 — Review Security Alerts

1. In Defender for Cloud, go to **Security alerts**
2. Filter by resource: your Service Bus namespace
3. Common alerts for Service Bus:
   - Unusual access patterns
   - Access from suspicious IP addresses
   - Unusual volume of operations

### Step 7 — Check Azure Policy Compliance

```bash
# List policy assignments related to Service Bus
az policy assignment list \
  --resource-group rg-servicebus-workshop \
  --query "[?contains(displayName, 'Service Bus')].{name: displayName, enforcement: enforcementMode}" \
  --output table
```

### Step 8 — Review the Secure Score Impact

1. In Defender for Cloud, check your **Secure Score**
2. Click on the score to see which Service Bus recommendations affect it
3. Each fixed recommendation improves your score

---

## Aviation Security Checklist for Service Bus

| Requirement | Setting | Where |
|---|---|---|
| Encrypt data in transit | TLS 1.2 minimum | Namespace → Properties |
| Encrypt data at rest | Service-managed or CMK | Premium only |
| Network isolation | Private endpoints | Premium → Networking |
| Authentication | Microsoft Entra ID (disable SAS) | Namespace → Properties |
| Audit logging | Diagnostic settings → Log Analytics | Monitoring → Diagnostic settings |
| Access control | RBAC roles, least privilege | Access control (IAM) |
| Threat detection | Defender for Cloud enabled | Subscription level |

---

## Key Takeaways

- Defender for Cloud provides continuous security assessment for Service Bus
- Aviation requires defense-in-depth: network isolation + authentication + encryption + monitoring
- Premium tier provides the most security features (Private Link, CMK, VNet)
- Disable SAS key authentication in production — use Microsoft Entra ID
- Enable diagnostic logging for security investigation and compliance

---

## Review Questions

1. Why should aviation systems use Private Endpoints instead of public access with IP filtering?
2. What is the risk of leaving SAS key authentication enabled?
3. How does disabling public network access affect Service Bus Explorer in the Portal?
