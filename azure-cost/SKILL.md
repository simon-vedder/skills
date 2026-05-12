---
name: azure-cost
description: Analyze Azure costs for existing resource groups or estimate pre-deploy costs from IaC files. Use when the user asks about Azure cost analysis, cost breakdown, how much their Azure resources cost, cost of a resource group, Terraform cost estimate, Bicep cost estimate, ARM template cost estimate, pre-deploy cost, Azure spend, reduce Azure costs, or cost optimization recommendations.
---

# Azure Cost Analysis

## Purpose
Act as an Azure cost analyst. Two modes:
- **`--live <resource-group>`** — analyze actual spend and surface cost optimization findings for an existing Azure resource group.
- **`--plan <file>`** — estimate pre-deploy costs from an IaC file (Terraform plan JSON, Bicep, or ARM template). Auto-detects format.

Prices always in USD at retail rates. Use Azure MCP pricing tools and Azure CLI for live data. Never invent prices or SKU mappings.

---

## Mode: `--live <resource-group>`

### Step 1 — Auth check
Run:
```bash
az account show --query "{subscription:name, id:id, tenant:tenantId}" -o table
```
If not signed in, stop and tell the user to run `az login`. State the subscription context before proceeding.

Warn: live mode requires `Microsoft.CostManagement/query/action` + `*/read` on the resource group. If cost query fails with 403, surface this permission note.

### Step 2 — Actual spend (last 30 days)
```bash
az costmanagement query \
  --type Usage \
  --scope "subscriptions/<sub-id>/resourceGroups/<rg>" \
  --dataset-aggregation '{"totalCost":{"name":"PreTaxCost","function":"Sum"}}' \
  --dataset-grouping '[{"type":"Dimension","name":"ServiceName"}]' \
  --timeframe MonthToDate \
  -o table
```
Extract: total spend, top cost drivers by service.

### Step 3 — Resource inventory
```bash
az resource list --resource-group <rg> --query "[].{name:name, type:type, location:location, sku:sku}" -o table
```

### Step 4 — Advisor cost recommendations
Use Azure MCP advisor tool:
```
intent: "Get cost recommendations for resource group <rg>"
command: recommendations list
```
Extract all recommendations with `category == "Cost"`. Note potential savings amounts where provided.

### Step 5 — Build output (see Output Format)

---

## Mode: `--plan <file>`

### Step 1 — Detect IaC format
Read the file and detect by top-level keys:
- **Terraform plan JSON**: has `"terraform_version"` key. Resources in `resource_changes[].change.after`.
- **Bicep compiled ARM**: has `"$schema"` containing `deploymentTemplate` and `"contentVersion"`. Resources in `resources[]`.
- **ARM template**: same as Bicep compiled output shape.

If format is ambiguous, ask the user.

### Step 2 — Extract priceable resources

**Terraform plan JSON:**
- Iterate `resource_changes[]` where `change.actions` contains `"create"` or `"update"`.
- For each: extract `type`, `change.after.location`, and SKU-related fields: `size`, `sku`, `tier`, `capacity`, `sku_name`, `vm_size`, `account_tier`, `account_replication_type`, `service_level`.
- Flag resources where SKU fields are `null` or present in `change.after_unknown` — these cannot be priced.

**ARM/Bicep:**
- Iterate `resources[]`.
- For each: extract `type`, `location`, `sku.name`, `sku.tier`, `sku.capacity`, `properties.hardwareProfile.vmSize`.
- Flag resources with no `sku` block and no known free-tier classification.

### Step 3 — Price each resource
For each priceable resource, call Azure MCP pricing tool:
```
intent: "Get retail price for <resource type> with SKU <sku> in region <location>"
command: get
parameters: { serviceName: "...", skuName: "...", armRegionName: "..." }
```
Capture `retailPrice` (hourly) and multiply: hourly × 730 = monthly estimate.

For resources with no direct pricing match, note "pricing not found — check Azure Pricing Calculator."

### Step 4 — Build output (see Output Format)

---

## Output Format

Use this structure for both modes:

```md
## Azure Cost Analysis — <resource-group or filename>

<One-sentence summary: total monthly spend or estimate, and the #1 cost driver.>

### Resource Cost Breakdown

| Resource | Type | SKU / Size | Region | Est. Monthly (USD) |
| --- | --- | --- | --- | --- |
| my-vm | Virtual Machine | Standard_D4s_v3 | eastus | $140.16 |
| my-storage | Storage Account | LRS / Standard | eastus | $4.80 |
| my-unknown-resource | App Service Plan | — (SKU unknown) | eastus | — |

**Total: ~$X.XX / month** *(retail rates, USD. Actual billed costs may differ due to reservations, EA discounts, or free tier credits.)*

### Cost Optimization Findings
[Live mode: Advisor recommendations with potential savings]
[Plan mode: architecture-level observations]

- **[Finding title]** — <explanation and recommended action> *(potential saving: $X/month)*

### Optimization Recommendations
- <Specific, actionable recommendation tailored to the resources found>
- <e.g. "Switch my-vm to a Reserved Instance (1yr) — ~40% saving on compute">
- <e.g. "my-storage: enable lifecycle management to tier cold blobs to Archive">
- <e.g. "Consider Spot instances for non-critical workloads in this RG">
```

Rules:
- Always include the disclaimer line about retail rates.
- Resources with unknown SKU: show `—` in both SKU and cost columns, note reason inline.
- If Cost Management query returns no data (new RG or insufficient permission): say so explicitly, do not estimate from inventory alone without noting it.
- For plan mode: prefix total with `~` and label it "estimated pre-deploy cost."

---

## Verification Sources

Use in this order:
1. Azure MCP pricing tool — retail prices per SKU and region.
2. Azure MCP advisor tool — cost recommendations scoped to subscription/RG.
3. Azure CLI:
   - `az costmanagement query` — actual billed spend.
   - `az resource list` — resource inventory with SKU data.
   - `az monitor metrics list` — utilization data when rightsizing recommendations are relevant.
4. Azure Pricing Calculator reference for services with complex pricing not covered by retail API.

Never invent a price. If a resource has no pricing match, flag it explicitly.

---

## Common Resource → SKU Mappings

| Terraform type | ARM type | SKU field |
| --- | --- | --- |
| `azurerm_linux_virtual_machine` | `Microsoft.Compute/virtualMachines` | `vm_size` / `hardwareProfile.vmSize` |
| `azurerm_windows_virtual_machine` | `Microsoft.Compute/virtualMachines` | `vm_size` |
| `azurerm_storage_account` | `Microsoft.Storage/storageAccounts` | `account_tier` + `account_replication_type` |
| `azurerm_app_service_plan` | `Microsoft.Web/serverfarms` | `sku_name` / `sku.name` |
| `azurerm_sql_database` | `Microsoft.Sql/servers/databases` | `sku_name` + `max_size_gb` |
| `azurerm_kubernetes_cluster` | `Microsoft.ContainerService/managedClusters` | node pool `vm_size` × `node_count` |
| `azurerm_cosmosdb_account` | `Microsoft.DocumentDB/databaseAccounts` | `offer_type` + RU/s from throughput settings |
| `azurerm_redis_cache` | `Microsoft.Cache/Redis` | `sku_name` + `family` + `capacity` |

For AKS: price each node pool separately. Total = sum of (vm_size price × node_count) across all pools.

---

## Free / Non-Priceable Resources

Flag these as `— (free tier / no retail price)` rather than attempting to price them:
- `azurerm_resource_group`, `Microsoft.Resources/resourceGroups`
- `azurerm_virtual_network`, `Microsoft.Network/virtualNetworks` (no per-unit charge)
- `azurerm_subnet`, `Microsoft.Network/virtualNetworks/subnets`
- `azurerm_role_assignment`, `Microsoft.Authorization/roleAssignments`
- Most `azurerm_*_policy*` resources
