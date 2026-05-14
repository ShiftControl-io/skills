# ShiftControl Skills

> Let your AI assistant manage your SaaS. These skills connect Claude, Cursor, ChatGPT, and other AI tools to [ShiftControl](https://shiftcontrol.io) so you can update subscription costs, audit spend, and reconcile your records with real invoices — through natural conversation, with you approving every change.

## What you can do today

- **Stop chasing invoices through your inbox.** Ask your AI to *"refresh my subscription info from my recent invoices"* and it'll search your email, match each invoice to an app in ShiftControl, show you exactly what would change (e.g. Slack $8/user/mo → $7/user/mo, contract through 2027-03-15), and update only what you approve. Each change records a short note in the app explaining where the new values came from.
- **Use the AI tool you already use.** Claude Desktop, Claude Code, Cursor, Windsurf, Cline, Continue.dev, ChatGPT, Gemini — see [INSTALL.md](INSTALL.md) for your tool.
- **Sign in once.** Connecting takes a single OAuth click through your normal ShiftControl login. Your AI gets exactly the permissions you already have — nothing more.

## Quick start

1. **Connect your AI to ShiftControl** — one-time MCP server install. [INSTALL.md](INSTALL.md) has step-by-step for every supported AI tool.
2. **Install a skill** — copy a workflow from the catalog below into your AI assistant. Claude surfaces (Desktop / Code / claude.ai) install natively; Cursor / Windsurf / ChatGPT accept the skill as a rule or system prompt.
3. **Just ask.** *"Refresh my ShiftControl subscriptions from my recent invoices."* Your AI follows the skill, calls ShiftControl on your behalf, shows you what would change, and waits for your green light.

## Available skills

| Skill | What it does | Status |
|---|---|---|
| [`refresh-subscription-info`](skills/refresh-subscription-info/) | Finds recent SaaS invoices in the user's email and proposes ShiftControl subscription updates (cost, billing frequency, contract terms, notes). | v0.1.0 |

More coming. See [open skill proposals](https://github.com/ShiftControl-io/skills/issues?q=label%3Askill-request).

## How skills work

Anthropic's [Claude Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) format is the source of truth: a folder containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and instructions in markdown. Anthropic surfaces (Claude Desktop, Claude Code, claude.ai, the Claude API) load skills natively. For other AI tools (Cursor, Windsurf, etc.), the `SKILL.md` content is copy-pasteable as a rule or system prompt — [INSTALL.md](INSTALL.md) has per-tool instructions.

Every skill in this repo follows three rules:

- **Assumes the ShiftControl MCP server is installed and authenticated** — see INSTALL.md
- **Proposes changes for human approval before any write** — no silent mutations
- **Records a short note on each change** — so the next person to look at the app's record can see where the updated values came from

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). **Signed commits are required.**

## License

[Apache 2.0](LICENSE) — use, fork, modify, redistribute freely with attribution.

## Support

- Bug reports and skill requests: [open an issue](https://github.com/ShiftControl-io/skills/issues/new/choose)
- Questions about ShiftControl: [support@shiftcontrol.io](mailto:support@shiftcontrol.io)
