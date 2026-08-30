# AKIJ Skills

Agent skills for the AKIJ Resource AI environment (OpenCode).

## Install

Requires the OpenCode agent. Recipients must already have the `ARL-DataMart_*` and
`ARL-Strategy_drive_*` MCP servers configured.

```bash
npx skills add akij-ai/skills/arl-business-case
```

Then restart OpenCode. The skill is discoverable automatically (it has a model-facing description).

## Skills

| Skill | What it does |
|---|---|
| `arl-business-case` | Builds a board-ready business case (HTML) for any AKIJ Resource SBU for any fiscal year, pulling actuals + budget from the DataMart MCP and strategy/budget documents from the Strategy MCP Drive library. |

## Requirements

- OpenCode (or any agent that supports `SKILL.md` skills)
- MCP servers: `ARL-DataMart` (BI DataMart, BU-scoped financials) and `ARL-Strategy` (shared "AR- 05 Years Strategy 27-31" Drive)
- Access to the shared AKIJ Drive root used by the Strategy MCP
