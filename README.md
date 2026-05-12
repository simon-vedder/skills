# Skills

A collection of custom skills for AI coding agents. Each skill extends an agent with focused domain expertise — loaded as a prompt extension so the agent can answer questions, analyze code, and generate outputs it otherwise couldn't do well.

Skills are designed for Claude Code and GitHub Copilot / Codex, but any agent that supports prompt-extension or custom instruction files can use them.

## What is a skill?

A skill is a self-contained directory with two files:

- `skill.md` — the agent loads this as a system prompt extension, giving it a specific persona, workflow, and reference knowledge for a domain.
- `README.md` — setup instructions, usage examples, and output reference for humans.

When an agent has a skill loaded, you can invoke it by name in plain language. The agent follows the skill's workflow rather than its general defaults.

## Available Skills

| Skill | Description | Agents | Requirements |
| --- | --- | --- | --- |
| [azure-rbac-advisor](./azure-rbac-advisor) | Least-privilege Azure RBAC advisor. Analyzes Terraform, Bicep, and ARM templates, maps operations to built-in roles, and generates custom role JSON. | Claude Code, Copilot | Optional: Azure MCP or Azure CLI for live tenant data |
| [azure-cost](./azure-cost) | Azure cost analysis. Two modes: analyze actual spend + Advisor findings for an existing resource group (`--live`), or estimate pre-deploy costs from Terraform/Bicep/ARM (`--plan`). | Claude Code, Copilot | Live mode: Azure CLI + Cost Management permissions. Plan mode: none |
| [azure-updates](./azure-updates) | Azure updates impact analyzer. Fetches Azure Updates RSS + Service Health, cross-references against all resources in your subscriptions, and surfaces impacted resources with action steps, deadlines, and severity. Also shows a news table for upcoming GA/preview announcements. | Claude Code, Copilot | Azure CLI + Azure MCP |

## Installation

Copy any skill directory into your agent's skills folder and restart the agent:

```bash
# Claude Code
cp -R ./<skill-name> ~/.claude/skills/

# GitHub Copilot / Codex
cp -R ./<skill-name> ~/.codex/skills/
```

## Usage

Once installed, invoke a skill by name in plain language:

```text
Use the azure-rbac-advisor skill to analyze ./infra for least-privilege Azure RBAC permissions.
```

Or just ask naturally — if the agent has the skill loaded, it will apply it automatically when relevant:

```text
What permissions do I need to deploy a VM with a user-assigned managed identity on an existing subnet?
```

See each skill's `README.md` for detailed usage examples and output reference.

## License

MIT — see [LICENSE](./LICENSE).

---

<sub>Built by Simon Vedder</sub>
