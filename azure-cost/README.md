# Azure Cost Analysis

This skill turns an agent into an Azure cost analyst. It works in two modes: analyze actual spend and surface optimization findings for an existing resource group, or estimate pre-deploy costs from an IaC file before anything is deployed.

The live mode queries Azure Cost Management for real billed spend, lists the resource inventory, and pulls Azure Advisor cost recommendations — so you get both the "how much are you spending" answer and the "here's what to cut" findings in one output. The plan mode reads a Terraform plan JSON, Bicep file, or ARM template, extracts resource SKUs, and prices each one against the Azure retail pricing API. Resources with unknown or dynamic SKUs are flagged inline rather than silently skipped.

All prices are shown in USD at retail rates.

## Setup

Copy this directory into your Claude or Codex skills folder:

```bash
cp -R ./azure-cost ~/.claude/skills/
# or for Codex
cp -R ./azure-cost ~/.codex/skills/
```

Restart the agent after copying so the skill is loaded.

## Usage

### Live mode — analyze an existing resource group

```text
Use azure-cost --live my-resource-group
```

Or in plain language:

```text
How much is the my-resource-group resource group costing me?
```

The skill will:
1. Confirm your signed-in subscription context.
2. Query Cost Management for month-to-date spend, broken down by service.
3. List all resources in the group with their SKUs.
4. Pull Azure Advisor cost recommendations for the subscription.
5. Return a narrative summary, resource cost table, and optimization findings.

**Required permissions:**
- `Microsoft.CostManagement/query/action` on the resource group scope
- `*/read` on the resource group

If Cost Management returns a 403, the skill will surface the missing permission rather than silently failing.

### Plan mode — estimate costs before deploying

```text
Use azure-cost --plan ./infra/tfplan.json
```

Or in plain language:

```text
Estimate the monthly cost of this Terraform plan before I deploy it.
```

Pass any of:
- `terraform show -json tfplan > tfplan.json` output (Terraform)
- A compiled Bicep ARM JSON (`bicep build main.bicep`)
- A raw ARM template JSON

The skill auto-detects the format and extracts resource types and SKUs. No Azure authentication required — pricing uses the public retail API.

## Data Sources

1. **Azure MCP pricing tool** — retail prices per SKU and region. Primary source for plan mode.
2. **Azure MCP advisor tool** — cost recommendations scoped to the subscription. Used in live mode.
3. **Azure CLI** — `az costmanagement query` for actual billed spend, `az resource list` for inventory. Used in live mode.

## Output

Both modes return the same structure:

### Summary line

One sentence stating the total monthly spend (live) or estimated pre-deploy cost (plan) and the top cost driver.

### Resource Cost Breakdown

| Resource | Type | SKU / Size | Region | Est. Monthly (USD) |
| --- | --- | --- | --- | --- |
| my-vm | Virtual Machine | Standard_D4s_v3 | eastus | $140.16 |
| my-storage | Storage Account | LRS / Standard | eastus | $4.80 |
| my-unknown | App Service Plan | — (SKU unknown) | eastus | — |

**Total: ~$X.XX / month** *(retail rates, USD. Actual billed costs may differ due to reservations, EA discounts, or free tier credits.)*

Resources with unknown or dynamic SKUs show `—` in both the SKU and cost columns with an inline reason. Free-tier resources (VNets, subnets, resource groups, role assignments) are flagged as `— (free tier / no retail price)`.

### Cost Optimization Findings

Live mode surfaces Azure Advisor cost recommendations with potential saving amounts where available.

Plan mode includes architecture-level observations based on the resources found (e.g. no Reserved Instance coverage, oversized SKUs relative to typical workloads).

### Optimization Recommendations

Actionable, resource-specific recommendations — for example:

- Switch `my-vm` to a 1-year Reserved Instance — ~40% saving on compute
- Enable lifecycle management on `my-storage` to tier cold blobs to Archive
- Consider Spot instances for non-critical workloads in this resource group
