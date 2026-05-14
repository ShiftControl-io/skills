# Install ShiftControl in your AI assistant

ShiftControl uses the [Model Context Protocol (MCP)](https://modelcontextprotocol.io) — an open standard for connecting AI assistants to applications. To use any of the skills in this repo, you first install the **ShiftControl MCP server**, then install the skill(s) in whichever way your AI tool supports.

You'll authenticate to ShiftControl once via PropelAuth (single sign-on, the same login you use for ShiftControl itself). The MCP server uses your account's permissions — anything you can do in ShiftControl, the AI can do on your behalf via this skill.

---

## Step 1 — Install the ShiftControl MCP server

### Claude Desktop (macOS / Windows / Linux)

Edit your Claude Desktop config:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux:** `~/.config/Claude/claude_desktop_config.json`

Add the `shiftcontrol` server inside `mcpServers`:

```json
{
  "mcpServers": {
    "shiftcontrol": {
      "command": "npx",
      "args": ["mcp-remote", "https://mcp.shiftcontrol.io/mcp"]
    }
  }
}
```

Restart Claude Desktop. The first time you use a ShiftControl tool, a browser tab opens for PropelAuth login. Approve once.

### Claude Code

```bash
claude mcp add shiftcontrol --transport http https://mcp.shiftcontrol.io/mcp
```

The first tool call triggers OAuth in your browser.

### Cursor

1. Open **Cursor Settings → MCP**
2. Click **Add new MCP Server**
3. Name: `shiftcontrol`
4. URL: `https://mcp.shiftcontrol.io/mcp`
5. Save and restart Cursor when prompted
6. First tool call triggers OAuth

### Windsurf

1. Open **Windsurf Settings → MCP Servers**
2. Add server with URL `https://mcp.shiftcontrol.io/mcp`
3. Restart Windsurf
4. First tool call triggers OAuth

### Cline (VS Code)

1. Open **Cline → Settings → MCP Servers**
2. Add server `shiftcontrol` with URL `https://mcp.shiftcontrol.io/mcp`
3. OAuth on first use

### Continue.dev

Edit `~/.continue/config.json`:

```json
{
  "experimental": {
    "modelContextProtocolServers": [
      {
        "transport": {
          "type": "http",
          "url": "https://mcp.shiftcontrol.io/mcp"
        },
        "name": "shiftcontrol"
      }
    ]
  }
}
```

### ChatGPT

ChatGPT's MCP support is rolling out. As of writing:

- **Custom GPTs** can call external APIs via Actions. You can use the ShiftControl OpenAPI directly: spec at `https://api.shiftcontrol.io/api-docs/openapi.json`. This is a different path than MCP — Skills in this repo target MCP-aware clients.
- **ChatGPT Desktop** is gaining MCP support; check OpenAI's release notes for current status.

### Gemini CLI

```bash
gemini mcp add shiftcontrol --url https://mcp.shiftcontrol.io/mcp
```

(Adjust to current Gemini CLI syntax.)

### Generic OAuth-MCP client

Any MCP client that supports remote OAuth 2.1 servers with Dynamic Client Registration works. Point it at:

- **Endpoint:** `https://mcp.shiftcontrol.io/mcp`
- **OAuth discovery:** `https://mcp.shiftcontrol.io/.well-known/oauth-authorization-server`

---

## Step 2 — Install a skill

### Claude Code (filesystem)

```bash
# Personal scope — available across all your Claude Code projects
mkdir -p ~/.claude/skills

# Sparse-clone just the skill you want
cd ~/.claude/skills
git clone --depth 1 --filter=blob:none --sparse https://github.com/ShiftControl-io/skills.git _tmp
cd _tmp && git sparse-checkout set skills/refresh-subscription-info
mv skills/refresh-subscription-info ../
cd .. && rm -rf _tmp
```

Claude Code discovers it on the next session.

### claude.ai (web)

1. Download the skill zip from the [latest release](https://github.com/ShiftControl-io/skills/releases/latest)
2. Open **claude.ai → Settings → Features → Skills**
3. Click **Upload skill** and select the zip
4. The skill is available immediately for new conversations

### Cursor / Windsurf / Cline (rules-based tools)

These tools use **rules** rather than skills. Paste the SKILL.md body as a rule:

1. Open the skill's `SKILL.md` on GitHub (use the raw view button)
2. Copy everything below the YAML frontmatter (the `---` block)
3. In **Cursor:** Settings → Rules → Add new rule → paste content, name it after the skill
4. In **Windsurf:** create `.windsurfrules` in the project root and paste content
5. In **Cline:** Settings → Custom Instructions → paste content

The AI follows the pasted instructions and calls the ShiftControl MCP tools from Step 1.

### ChatGPT (Custom GPT)

Create a Custom GPT with the SKILL.md body as system instructions. Add the ShiftControl OpenAPI (`https://api.shiftcontrol.io/api-docs/openapi.json`) as an Action. The Custom GPT then has both workflow logic and API access.

---

## Troubleshooting

### "Could not connect to MCP server"

- Confirm the URL is `https://mcp.shiftcontrol.io/mcp` (note: `.io`, `https://`, ends in `/mcp`)
- If using `npx mcp-remote`, ensure Node 20+ is installed
- Open `https://mcp.shiftcontrol.io/.well-known/oauth-authorization-server` in a browser — it should return JSON

### "OAuth flow doesn't complete"

- Cookies / popup blockers may interfere — try a different browser
- If your AI assistant runs in a sandbox without browser access, you may need a different MCP client

### "Authentication succeeded but tools don't appear"

- Restart the AI assistant after the first auth
- Confirm the user account on `app.shiftcontrol.io` has at least one organization membership

### "Tools work but I get permission errors on writes"

The MCP server respects ShiftControl's existing role-based access. If your user can't perform an action in the ShiftControl UI, the MCP server won't either. Ask your ShiftControl admin to grant the needed role.

---

## Need help?

- Bug reports: [github.com/ShiftControl-io/skills/issues](https://github.com/ShiftControl-io/skills/issues)
- Account / access questions: [support@shiftcontrol.io](mailto:support@shiftcontrol.io)
