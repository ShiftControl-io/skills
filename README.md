# ShiftControl Skills

> Customer-facing AI skills (instruction sets) for [ShiftControl](https://shiftcontrol.io). Each skill pairs with the ShiftControl MCP server at `https://mcp.shiftcontrol.io/mcp` to drive a specific SaaS-management workflow.

## Quick start

1. **Install the ShiftControl MCP server** in your AI assistant. See [INSTALL.md](INSTALL.md) for per-platform instructions (Claude Desktop, Claude Code, Cursor, Windsurf, ChatGPT, and more).
2. **Install one or more skills** from the table below — each is a folder with a `SKILL.md` your AI assistant loads on demand.
3. **Ask your AI to do the thing**, e.g. *"Refresh my ShiftControl subscription info from my recent invoices."*

## Available skills

| Skill | What it does | Status |
|---|---|---|
| [`refresh-subscription-info`](skills/refresh-subscription-info/) | Finds recent SaaS invoices in the user's email and proposes ShiftControl subscription updates (cost, billing frequency, contract terms, audit notes). | v0.1.0 |

More coming. See [open skill proposals](https://github.com/ShiftControl-io/skills/issues?q=label%3Askill-request).

## How skills work

Anthropic's [Claude Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) format is the source of truth: a folder containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and instructions in markdown. Anthropic surfaces (Claude Desktop, Claude Code, claude.ai, the Claude API) load skills natively. For other AI tools (Cursor, Windsurf, etc.), the `SKILL.md` content is copy-pasteable as a rule or system prompt — [INSTALL.md](INSTALL.md) has per-tool instructions.

Every skill in this repo follows three rules:

- **Assumes the ShiftControl MCP server is installed and authenticated** — see INSTALL.md
- **Proposes changes for human approval before any write** — no silent mutations
- **Logs an audit-friendly `notes` value on every change** — the change is traceable back to the skill, version, and source data

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). **Signed commits are required.**

## License

[Apache 2.0](LICENSE) — use, fork, modify, redistribute freely with attribution.

## Support

- Bug reports and skill requests: [open an issue](https://github.com/ShiftControl-io/skills/issues/new/choose)
- Questions about ShiftControl: [support@shiftcontrol.io](mailto:support@shiftcontrol.io)
