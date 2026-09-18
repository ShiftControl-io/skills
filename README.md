# ShiftControl Skills

> Let your AI assistant manage your SaaS. These skills connect Claude, Cursor, ChatGPT, and other AI tools to [ShiftControl](https://shiftcontrol.io) so you can update subscription costs, audit spend, and reconcile your records with real invoices — through natural conversation, with you approving every change.

## What you can do today

- **Stop chasing invoices through your inbox.** Ask your AI to *"refresh my subscription info from my recent invoices"* and it'll search your email **and your Xero bills**, match each invoice to an app in ShiftControl, show you exactly what would change (e.g. Slack $8/user/mo → $7/user/mo, contract through 2027-03-15), and update only what you approve. Each change records a short note in the app explaining where the new values came from.
- **Ask someone else to find them.** If the invoices live in the bookkeeper's Xero or finance's inbox, your AI writes a request they can run themselves. They send back one file; you review the proposed changes as usual. They never need a ShiftControl login, and nothing is written on their side.
- **Use the AI tool you already use.** Claude Desktop, Claude Code, Cursor, Windsurf, Cline, Continue.dev, ChatGPT, Gemini — see [INSTALL.md](INSTALL.md) for your tool.
- **Sign in once.** Connecting takes a single OAuth click through your normal ShiftControl login. Your AI gets exactly the permissions you already have — nothing more.

## Quick start

1. **Connect your AI to ShiftControl** — one-time MCP server install. [INSTALL.md](INSTALL.md) has step-by-step for every supported AI tool.
2. **Install the skills** — in Claude, add this repo as a plugin marketplace and install **ShiftControl**. Everywhere else, `npx skills add ShiftControl-io/skills`. Zip downloads and copy-paste still work; [INSTALL.md](INSTALL.md) has every route.
3. **Just ask.** *"Refresh my ShiftControl subscriptions from my recent invoices."* Your AI follows the skill, calls ShiftControl on your behalf, shows you what would change, and waits for your green light.

## Available skills

| Skill | What it does | Status |
|---|---|---|
| [`refresh-subscription-info`](skills/refresh-subscription-info/) | Finds SaaS invoices — in email, in Xero bills, or in a file a colleague collected — and proposes ShiftControl subscription updates (cost, billing frequency, contract terms, notes). | v0.3.0 |

More coming. See [open skill proposals](https://github.com/ShiftControl-io/skills/issues?q=label%3Askill-request).

## How skills work

[Agent Skills](https://agentskills.io/specification) is the format, and it is an open one: a folder containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and instructions in markdown. Claude, OpenAI Codex, Gemini CLI, Devin and Cursor all read it, and `.agents/skills/` is the shared directory convention between them.

Distribution is the part every vendor still does its own way, so this repo carries all of them at once:

- **Claude plugin marketplace** — `.claude-plugin/marketplace.json` makes the repo installable as one plugin in Claude Code, the Claude desktop app and claude.ai
- **Vendor-neutral installer** — `npx skills add ShiftControl-io/skills` writes the skills into whichever directory your agent uses
- **Git and `.agents/skills/`** — clone a skill folder straight into a repo for Devin, Codex or Gemini CLI
- **Zip per skill** — attached to every [release](https://github.com/ShiftControl-io/skills/releases/latest) for drag-and-drop upload

[INSTALL.md](INSTALL.md) has the exact command for each.

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
