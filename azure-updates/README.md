# Azure Updates Impact Analyzer

This skill fetches Azure service announcements from the **Microsoft Release Communications MCP Server**, cross-references them against all resources in your Azure subscriptions, and surfaces only the updates that actually affect your environment — with concrete action steps, deadlines, and source links.

## Prerequisites

**Microsoft Release Communications MCP Server** — public, free, no auth required.

Add it to Claude Code once:

```bash
claude mcp add --transport http mrc-mcp https://www.microsoft.com/releasecommunications/mcp
```

Restart Claude Code after adding.

**Azure CLI** — signed in (`az login`). Azure MCP configured for best results.

## Setup

Copy this directory into your Claude or Codex skills folder:

```bash
cp -R ./azure-updates ~/.claude/skills/
# or for Codex
cp -R ./azure-updates ~/.codex/skills/
```

Restart the agent after copying so the skill is loaded.

## Usage

```text
azure-updates
```

Check last 30 days of updates against your environment.

```text
azure-updates --days 14
```

Check last 14 days instead.

## Data Sources

1. **Microsoft Release Communications MCP** (`get_recent_azure_updates`) — primary source. Queries Azure Updates by tag (Retirements, Security, Pricing, GA, Preview, In Development). No RSS, no timeout issues.
2. **Azure MCP** — resource inventory across all accessible subscriptions for environment cross-referencing.
3. **Azure CLI** — fallback for resource listing.

## Output

The skill returns two tables.

### Impact Table

Retirements, security advisories, and pricing changes filtered to only the updates that match resources in your environment. Non-matching updates are discarded.

| Severity | Update | Category | Affected Resources | Deadline | Action Steps | Source |
| --- | --- | --- | --- | --- | --- | --- |
| 🔴 Critical | AKS — TLS 1.0/1.1 retirement | Security | `prod-aks` (rg-prod, Sub A) | 2026-03-31 | 1. Update TLS config... | [Link] |
| 🟠 High | Azure SQL Basic tier retirement | Retirement | `db-legacy` (rg-data, Sub A) | 2026-09-30 | 1. Migrate to General Purpose... | [Link] |

Severity levels:
- 🔴 **Critical** — retirement deadline ≤ 90 days, or active security incident
- 🟠 **High** — retirement deadline > 90 days, or security advisory
- 🟡 **Medium** — pricing change
- 🔵 **Low** — other impacting update

### News Table

GA releases, previews, and in-development features from the same time window. Not filtered by your environment.

| Update | Category | Summary | Source |
| --- | --- | --- | --- |
| Azure Container Apps dedicated plan | Generally Available | Dedicated plan GA in all regions | [Link] |
| Azure Policy new built-ins | Public Preview | 15 new network security definitions | [Link] |
