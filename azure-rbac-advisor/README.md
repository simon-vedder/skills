# Azure RBAC Advisor

This skill turns an agent into a least-privilege Azure RBAC advisor. It answers natural-language permission questions, inspects Terraform, Bicep, and ARM templates to identify required control-plane operations, maps those operations to built-in roles ranked by least privilege, and generates custom role JSON when no built-in role fits cleanly.

The advisor separates permissions by target scope — deployment scope, resource creation scope, and linked-resource scope (e.g. joining an existing subnet or assigning an existing managed identity) — so role assignments stay as narrow as possible. It avoids recommending Contributor or Owner unless deployment simplicity is explicitly more important than least privilege.

## Setup

Copy this directory into your Claude or Codex skills folder:

```bash
cp -R ./azure-rbac-advisor ~/.claude/skills/
# or for Codex
cp -R ./azure-rbac-advisor ~/.codex/skills/
```

Restart the agent after copying so the skill is loaded.

## Usage

Ask for Azure RBAC analysis in plain language, for example:

```text
What permissions do I need to create a VM with a user-assigned managed identity on an existing subnet?
```

Or ask it to inspect IaC:

```text
Use the azure-rbac-advisor skill to analyze ./infra for least-privilege Azure RBAC permissions and generate a custom_role.json.
```

## Data Sources

The skill uses three source tiers in order of preference:

1. **Azure MCP** — when your agent has Azure MCP configured, the skill queries it directly for live role definitions, provider operation lists, and current subscription context. This is the most accurate source and preferred for tenant-specific validation.

2. **Azure CLI** — used when MCP is unavailable. Sign in first for tenant-aware results:

   ```bash
   az login
   az account set --subscription <subscription-id>
   ```

   The skill will run commands like `az provider operation list` and `az role definition list` to verify operation strings against your tenant.

3. **Microsoft Learn** — Azure built-in roles reference, Azure permissions by resource provider, and the custom role definition schema. Used as fallback for static IaC review when no live Azure access is available.

The skill works without any Azure access for static IaC analysis, but live data increases confidence from `Medium` to `High` for role and operation validation.

## Output

The skill returns a structured analysis with four sections:

### Required Actions

A table of every Azure control-plane operation implied by the request or IaC, grouped by target scope:

| Scope | Why | Actions |
| --- | --- | --- |
| Resource group | Deploy VM | `Microsoft.Compute/virtualMachines/write` |
| Existing subnet | Attach NIC | `Microsoft.Network/virtualNetworks/subnets/join/action` |

### Best Built-In Role Assignments

Roles ranked by least privilege, from narrowest service-specific operator to broadest Contributor. Each entry includes what the role covers and its tradeoff:

| Rank | Role | Scope | Covers | Tradeoff |
| --- | --- | --- | --- | --- |
| 1 | Virtual Machine Contributor | Resource group | VM compute | Doesn't cover NIC or identity |
| 2 | Managed Identity Operator | Specific identity | Assign existing UAMI | Not needed if UAMI is created here |

The advisor calls out when no built-in role fits well and a custom role is cleaner.

### Custom Role (when requested)

A complete Azure CLI / PowerShell-compatible role definition JSON scoped to the narrowest acceptable operation list:

```json
{
  "Name": "Least Privilege Deployment Operator",
  "Description": "...",
  "Actions": ["Microsoft.Resources/deployments/*"],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": ["/subscriptions/<id>/resourceGroups/<rg>"]
}
```

### Notes

- **Assumptions** — scope, existing vs. new resources, deployment method
- **Verification** — confidence level (`High` = verified from live data or current docs, `Medium` = inferred from stable operation naming, `Low` = provider behavior ambiguous) and which sources were used
- **Gaps** — operations that could not be confirmed or depend on runtime behavior

## Applying Role Assignments

After the advisor produces its output, you can tell the skill to apply the recommended assignments directly to your tenant. The skill will prompt for a principal, show a full manifest of all changes, ask for confirmation, and then execute.

**Prerequisites:** Azure CLI installed and signed in with sufficient permissions to create role definitions and role assignments at the target scope (typically `User Access Administrator` or `Owner`).

```text
Apply the recommended role assignments to john@contoso.com
```

The skill will:

1. Ask for a principal (UPN like `john@contoso.com` or object ID GUID) if not already provided.
2. Show every change that will be made before touching anything:

   ```
   Will create the following in Azure:

     [Custom Role]  "Least Privilege Deploy Operator"  AssignableScopes: /subscriptions/...
     [Assignment]   "Least Privilege Deploy Operator"  Scope: /subscriptions/.../resourceGroups/myRG  →  john@contoso.com
     [Assignment]   "Managed Identity Operator"        Scope: .../identities/myId                    →  john@contoso.com

   Confirm? (yes / no)
   ```

3. On explicit `yes`: create any custom role first, then all assignments. Failures are reported per-row without aborting the batch.
4. Return a summary table of every created or failed item.

You can also override the target scope at apply time if you want assignments at a different scope than the advisor recommended.
