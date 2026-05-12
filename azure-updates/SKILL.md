---
name: azure-updates
description: Fetch Azure service updates and match them against your Azure environment. Surface impacted resources with action steps, deadlines, and source links. Use when the user wants to check Azure updates, retirement notices, security advisories, pricing changes, or upcoming feature announcements affecting their environment.
---

# Azure Updates Impact Analyzer

## Purpose
Fetch Azure service updates from the Microsoft Release Communications MCP Server, cross-reference against all resources across all Azure subscriptions, and produce:
1. **Impact table** — retirements, security advisories, and pricing changes that affect your actual environment, with LLM-generated action steps and deadlines.
2. **News table** — upcoming GA, preview, and in-development announcements (informational, not filtered by environment).

## Invocation
- `azure-updates` — check last 30 days (default)
- `azure-updates --days <N>` — check last N days

---

## Prerequisites

This skill requires the **Microsoft Release Communications MCP Server** configured in Claude Code. If not yet added, run once:

```bash
claude mcp add --transport http mrc-mcp https://www.microsoft.com/releasecommunications/mcp
```

Then restart Claude Code. No authentication required — the server is public and free.

---

## Core Workflow

### Step 1 — Auth check

```bash
az account show --query "{subscription:name, id:id, tenant:tenantId}" -o table
```

If not signed in, stop and tell the user to run `az login`. State active tenant before proceeding.

---

### Step 2 — Discover Azure environment

List all enabled subscriptions:

```bash
az account list --query "[?state=='Enabled'].{name:name, id:id}" -o json
```

For each subscription, list all resources using Azure MCP group and resource tools, or Azure CLI:

```bash
az resource list --subscription <sub-id> \
  --query "[].{name:name, type:type, resourceGroup:resourceGroup, location:location}" \
  -o json
```

Collect full inventory as: `name | ARM type | resourceGroup | subscription`. If a subscription returns an access error, note it and continue.

---

### Step 3 — Fetch impact updates (Retirements, Security, Pricing)

Use the MRC MCP `get_recent_azure_updates` tool to fetch updates filtered by relevant tags. Run separate queries for each tag to maximize results:

**Retirements:**
```
tool: get_recent_azure_updates
filter: tags eq 'Retirements'
search: (leave empty)
top: 50
```

**Security:**
```
tool: get_recent_azure_updates
filter: tags eq 'Security'
top: 50
```

**Pricing:**
```
tool: get_recent_azure_updates
filter: tags eq 'Pricing & Offerings'
top: 50
```

After fetching, filter results client-side to only items published within the configured time window (default: last 30 days). Deduplicate across queries by update ID.

Combine all results into the **impact bucket**.

---

### Step 4 — Fetch news updates (GA, Preview, In Development)

Use MRC MCP `get_recent_azure_updates` for each status type:

**Generally Available:**
```
tool: get_recent_azure_updates
filter: tags eq 'Generally Available'
top: 50
```

**Preview:**
```
tool: get_recent_azure_updates
filter: tags eq 'Public Preview'
top: 50
```

**In Development:**
```
tool: get_recent_azure_updates
filter: tags eq 'In development'
top: 50
```

Filter to configured time window. Combine into **news bucket**.

---

### Step 5 — Semantic matching (impact bucket only)

For each item in the impact bucket:

1. Read title and description (use `get_azure_update_by_id` to fetch full description if needed for action step generation).
2. Identify the Azure service(s) mentioned.
3. Map to ARM resource type(s): use the mapping table at the bottom as a reference, but also reason directly against the ARM types present in the actual inventory. ARM type strings are human-readable — use semantic judgment for any service not in the table.
4. Check resource inventory — any matching resources?
5. **Match found:** add to impacted set with specific resource names, resource groups, and subscriptions.
6. **No match:** discard — do not show in impact table.

For each matched update extract:
- **Deadline:** any retirement/end-of-life date from the description. Leave blank if none.
- **Action steps:** generate 1–3 concrete steps based on description content (e.g. "Migrate `prod-aks` from API version X to Y before [date]").
- **Severity:** `Critical` if deadline ≤ 90 days or active security incident; `High` if deadline > 90 days or security advisory; `Medium` for pricing; `Low` for other.

---

### Step 6 — Generate output

See Output Format below.

---

## Output Format

```md
## Azure Updates Impact Report
*Checked: <today's date> | Window: last <N> days | Subscriptions: <count> | Source: Microsoft Release Communications MCP*

---

### Impact Table
*Retirements, security advisories, and pricing changes affecting your environment.*

| Severity | Update | Category | Affected Resources | Deadline | Action Steps | Source |
| --- | --- | --- | --- | --- | --- | --- |
| 🔴 Critical | Azure Kubernetes Service — TLS 1.0/1.1 retirement | Security | `prod-aks` (rg-prod, Sub A) | 2026-03-31 | 1. Update TLS config to 1.2+. 2. Audit client apps. | [Link](https://...) |
| 🟠 High | Azure SQL Database Basic tier retirement | Retirement | `db-legacy` (rg-data, Sub A) | 2026-09-30 | 1. Migrate to General Purpose tier. 2. Update connection strings. | [Link](https://...) |
| 🟡 Medium | Azure Blob Storage pricing change in West Europe | Pricing | `staccountprod` (rg-storage, Sub A) | — | 1. Review cost impact. 2. Evaluate Archive tier for cold data. | [Link](https://...) |

*Updates with no matching resources in your environment are not shown.*

---

### News Table
*Upcoming GA releases, previews, and in-development features. Not filtered by your environment.*

| Update | Category | Summary | Source |
| --- | --- | --- | --- |
| Azure Container Apps — dedicated plan GA | Generally Available | Dedicated plan now GA in all regions. | [Link](https://...) |
| Azure Policy — new built-in definitions | Public Preview | 15 new built-in definitions for network security. | [Link](https://...) |

---

### Summary
- **<N> updates fetched** (last <N> days)
- **<N> impact your environment** across <N> subscriptions
- **<N> critical / <N> high / <N> medium severity**
- **<N> news items** (GA / Preview / In Development)

*Source: [Microsoft Release Communications MCP](https://www.microsoft.com/releasecommunications/mcp)*
```

---

## Severity Legend

| Emoji | Level | Condition |
| --- | --- | --- |
| 🔴 | Critical | Retirement deadline ≤ 90 days, or active security incident |
| 🟠 | High | Retirement deadline > 90 days, or security advisory |
| 🟡 | Medium | Pricing change |
| 🔵 | Low | Other impacting update |

---

## Verification Sources

1. **MRC MCP** `get_recent_azure_updates` — primary source for all Azure update data (retirements, security, pricing, GA, preview)
2. **MRC MCP** `get_azure_update_by_id` — fetch full description for specific updates when generating action steps
3. **Azure MCP** group and resource tools — environment discovery across subscriptions
4. **Azure CLI** — fallback for resource listing if Azure MCP unavailable

---

## Common ARM Type Mappings

| Azure Service | ARM resource type |
| --- | --- |
| Azure Kubernetes Service | `Microsoft.ContainerService/managedClusters` |
| Azure SQL Database | `Microsoft.Sql/servers/databases` |
| Azure SQL Server | `Microsoft.Sql/servers` |
| Azure Storage / Blob Storage | `Microsoft.Storage/storageAccounts` |
| Azure Virtual Machines | `Microsoft.Compute/virtualMachines` |
| Azure App Service | `Microsoft.Web/sites` |
| Azure App Service Plan | `Microsoft.Web/serverfarms` |
| Azure Functions | `Microsoft.Web/sites` (kind: functionapp) |
| Azure Container Apps | `Microsoft.App/containerApps` |
| Azure Key Vault | `Microsoft.KeyVault/vaults` |
| Azure API Management | `Microsoft.ApiManagement/service` |
| Azure Service Bus | `Microsoft.ServiceBus/namespaces` |
| Azure Event Hubs | `Microsoft.EventHub/namespaces` |
| Azure Cosmos DB | `Microsoft.DocumentDB/databaseAccounts` |
| Azure Redis Cache | `Microsoft.Cache/Redis` |
| Azure PostgreSQL | `Microsoft.DBforPostgreSQL/servers` or `flexibleServers` |
| Azure MySQL | `Microsoft.DBforMySQL/servers` or `flexibleServers` |
| Azure Virtual Network | `Microsoft.Network/virtualNetworks` |
| Azure Load Balancer | `Microsoft.Network/loadBalancers` |
| Azure Application Gateway | `Microsoft.Network/applicationGateways` |
| Azure Log Analytics | `Microsoft.OperationalInsights/workspaces` |
| Azure Container Registry | `Microsoft.ContainerRegistry/registries` |
| Azure Logic Apps | `Microsoft.Logic/workflows` |
| Azure Document Intelligence | `Microsoft.CognitiveServices/accounts` |
| Azure AI Services | `Microsoft.CognitiveServices/accounts` |

This table is a reference aid for common services — not an exhaustive lookup. For any service not listed, reason directly from the ARM type strings in the actual resource inventory. ARM types like `Microsoft.Batch/batchAccounts` or `Microsoft.Network/privateDnsZones` are human-readable enough to match semantically against the service name in an update. Never skip a potential match just because a service isn't in this table — if the update mentions a service and your inventory contains a resource whose ARM type plausibly maps to it, treat it as a match and include it.
