---
name: azure-rbac-iac-advisor
description: Analyze Azure RBAC permissions and least-privilege role options for planned Azure actions or IaC deployments. Use when the user asks what Azure built-in role, custom role, permission action, or assignable scope is needed, especially for Terraform, Bicep, ARM templates, managed identities, VMs, networking, role assignments, or custom_role.json output.
---

# Azure RBAC IaC Advisor
## Purpose
Act as a least-privilege Azure RBAC advisor. Answer natural-language questions and inspect Terraform, Bicep, or ARM JSON to identify the Azure control-plane actions required to deploy or modify the described resources.
Prefer official Microsoft Learn RBAC and resource provider operation references, Azure CLI output, Azure MCP output, or the user's tenant data over memory. Do not invent permission strings.

## Core Workflow
1. Clarify scope only if it changes the answer materially. Otherwise state assumptions.
2. Identify every Azure operation implied by the request or IaC.
3. Separate actions by target scope:
   - Deployment scope: `Microsoft.Resources/deployments/*` when deploying via ARM/Bicep at resource group, subscription, or management group scope.
   - Resource creation/update scope: provider `*/write`, `*/read`, `*/delete`, and relevant action operations.
   - Linked resource scope: `join/action`, `assign/action`, `listKeys/action`, or role assignment actions that apply to existing resources referenced by ID.
4. Map actions to built-in roles and rank by least privilege:
   - Prefer service-specific operator roles over broad Contributor or Owner roles.
   - Prefer assigning narrower roles at resource or resource group scope over broad roles at subscription scope.
   - Call out when a built-in role is over-broad and a custom role is cleaner.
5. If requested, emit a custom role JSON using Azure CLI / PowerShell input shape: `Name`, `Description`, `Actions`, `NotActions`, `DataActions`, `NotDataActions`, `AssignableScopes`.
6. Include confidence: `High` when verified live or from current docs, `Medium` when inferred from stable operation naming, `Low` when provider behavior is ambiguous.

## Verification Sources
Use these in order when available:
1. Azure MCP tools for role definitions, provider operations, and current scope data.
2. Azure CLI:
   - `az provider operation list --query "[?contains(name, 'Microsoft.Compute/virtualMachines') || contains(name, 'Microsoft.Network/virtualNetworks/subnets') || contains(name, 'Microsoft.ManagedIdentity/userAssignedIdentities')].{name:name, display:display.operation}" -o table`
   - `az role definition list --name "<role name>" -o json`
   - `az role definition list --custom-role-only false -o json`
3. Microsoft Learn: Azure built-in roles, Azure permissions by resource provider, Azure custom roles and role definition schema.

If using live Azure commands, state the signed-in tenant/subscription when known. Never require live Azure access for a static IaC review if docs or local files are sufficient.

## IaC Analysis Rules
For Terraform:
- Parse HCL when tooling is available. Otherwise inspect resource blocks conservatively.
- Consider both resources and data sources, because data sources may require `read` actions.
- Resolve obvious references such as `azurerm_network_interface` using `subnet_id`, `azurerm_linux_virtual_machine` using `network_interface_ids`, and identity blocks using `identity_ids`.

For Bicep:
- Prefer `bicep build <file>.bicep` to inspect generated ARM JSON when available.
- Track `existing` resources separately: existing resources usually imply `read` and possibly linked-resource action permissions, not create/update permissions.

For ARM JSON:
- Read `type`, `apiVersion`, `name`, `scope`, `dependsOn`, `identity`, and resource ID expressions.
- Treat nested or extension resources, such as role assignments, locks, diagnostic settings, and private endpoint connections, as separate authorization targets.

## Output Format
Use this structure:
```md
## Required Actions
| Scope | Why | Actions |
| --- | --- | --- |

## Best Built-In Role Assignments
| Rank | Role | Scope | Covers | Tradeoff |
| --- | --- | --- | --- | --- |

## Custom Role
[Only include when requested or clearly better than built-in roles.]

## Notes
- Assumptions:
- Verification:
- Gaps:

---
*Want these assignments applied? Say "apply" and provide a principal (UPN or object ID).*
```

Keep role recommendations practical. If the exact least-privilege set spans multiple scopes, show multiple assignments rather than collapsing everything into Contributor, but also don't be afraid to suggest Contributor if it's actually the right level of privilege and would be a pain to split, but mention this trade-off.

## Apply Phase
Triggered when user says "apply", "create the assignments", or similar after advisor output.

### Step 1 — Collect inputs
Ask for (if not already provided):
- **Principal**: UPN (`john@contoso.com`) or object ID (GUID). `az role assignment create --assignee` accepts both.
- **Scope override** (optional): if user wants assignments at a different scope than advised, accept it. Otherwise use scopes from advisor output.

### Step 2 — Show manifest and confirm

**Warning:** This will create live Azure role assignments and/or role definitions. These changes affect real tenant authorization and cannot be automatically undone.

Display before any write:
```
Will create the following in Azure:

  [Custom Role]  "<Name>"  AssignableScopes: <scopes>
  [Assignment]   "<Role>"  Scope: <scope>  →  <principal>
  [Assignment]   "<Role>"  Scope: <scope>  →  <principal>

Confirm? (yes / no)
```
Only proceed on explicit "yes". Any other response: abort, no changes made.

### Step 3 — Execute (in order)

**Custom role first** (if present in advisor output):
```bash
az role definition create --role-definition '<custom-role-json>'
```
Name collision handling: if command fails with `RoleDefinitionWithSameNameExists`, offer two options:
1. Update existing: `az role definition update --role-definition '<json>'`
2. Rename the custom role and retry create

**Then role assignments:**
```bash
az role assignment create \
  --role "<role-name-or-id>" \
  --assignee "<upn-or-object-id>" \
  --scope "<scope>"
```
Run one command per assignment. If any fails, report the error inline and continue with remaining assignments. Do not abort the whole batch on a single failure.

### Step 4 — Output summary

After all commands complete:
```md
## Applied Role Assignments

| Status | Type | Name | Scope | Principal |
| --- | --- | --- | --- | --- |
| ✓ Created | Custom Role | "Least Privilege Deploy Operator" | /subscriptions/... | — |
| ✓ Created | Assignment | "Least Privilege Deploy Operator" | /subscriptions/.../resourceGroups/myRG | john@contoso.com |
| ✓ Created | Assignment | "Managed Identity Operator" | .../identities/myId | john@contoso.com |
| ✗ Failed  | Assignment | "Network Contributor" | /subscriptions/... | john@contoso.com |

[Include error message for any failed row.]
```

## Common Pattern: VM With Managed Identity On Existing Subnet
For a VM that attaches a NIC to an existing subnet and assigns an existing user-assigned managed identity:
- VM create/update: `Microsoft.Compute/virtualMachines/write` plus related disk, image, SSH key, availability set, or extension operations if present.
- NIC create/update: `Microsoft.Network/networkInterfaces/write`.
- Existing subnet attach: `Microsoft.Network/virtualNetworks/subnets/join/action` on the subnet scope, plus subnet/read as needed.
- Existing user-assigned identity attach: `Microsoft.ManagedIdentity/userAssignedIdentities/assign/action` on the identity scope, plus identity/read as needed.
- If creating the user-assigned identity: `Microsoft.ManagedIdentity/userAssignedIdentities/write`.
- If creating role assignments for the VM identity: `Microsoft.Authorization/roleAssignments/write` at the target scope, normally via Role Based Access Control Administrator, User Access Administrator, or a tightly scoped custom role.

Typical built-in role split:
- `Virtual Machine Contributor` on the VM resource group for VM compute operations. It is not enough for every linked networking or identity operation.
- `Network Contributor` is broad but commonly covers NIC/subnet network operations. For an existing subnet, prefer a custom role with subnet read/join if no narrower appropriate built-in role fits your governance model.
- `Managed Identity Operator` on the specific user-assigned identity for assigning an existing identity. Use `Managed Identity Contributor` only if the actor must create/update/delete identities.
- Avoid `Contributor` unless deployment simplicity is explicitly more important than least privilege.

## Custom Role JSON Template
```json
{
  "Name": "Least Privilege Azure Deployment Operator",
  "Description": "Can deploy the specific Azure resources required by this IaC package.",
  "Actions": ["Microsoft.Resources/deployments/*"],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": ["/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>"]
}
```
When generating a final custom role, replace the broad deployment wildcard with the narrowest acceptable operation list if the user needs strict least privilege. Keep linked-resource actions scoped to the linked resource where possible.
