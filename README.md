# Grove — AI Plugin

Connect Claude, Codex, or Cursor to your firm in [Grove](https://www.grove.tax). Once installed, your AI assistant can list authorized returns, summarize the fact-sheet fields exposed through MCP, draft missing-document follow-ups, and import Drake / Lacerte backups — all scoped to your firm via OAuth (no API keys to paste).

> **Prerequisite for all install paths:** your firm needs MCP access enabled. Contact Grove support (`support@grove.tax`) to turn it on for your firm.

## Install in Claude Code (bundled — connector + skill in one step)

This repo is a Claude Code plugin marketplace. One command adds the marketplace; one more installs Grove (skill + MCP connector together):

```bash
/plugin marketplace add grove-tax/grove-mcp
/plugin install grove@grove
```

After install, run `/reload-plugins`. Grove's MCP tools and the `grove` skill are now available. First tool call triggers OAuth in your browser.

## Install in Claude.ai / Claude Desktop

Claude.ai and the Desktop app don't have a bundled "connector + skill" install path today. You install the two pieces separately:

### 1. Connector (required)

1. Go to **Settings → Connectors → Add custom connector** ([direct link](https://claude.ai/settings/connectors?modal=add-custom-connector))
2. Paste the URL: `https://app.grove.tax/api/mcp`
3. Name it `grove`
4. Save. Claude opens a browser for OAuth sign-in.

### 2. Skill (recommended)

Upload `plugins/grove/skills/grove/` from this repo via **Settings → Capabilities → Skills** in Claude.ai or Claude Desktop. The skill teaches Claude *how* to use the connector well (when to import vs. read, how Drake and Lacerte backups differ, how to phrase a client follow-up email). Without the skill the connector still works, but Claude will ask more clarifying questions.

## Install in Codex (bundled — connector + skill in one step)

This repo is also a Codex plugin marketplace. One command:

```bash
codex plugin marketplace add https://github.com/grove-tax/grove-mcp.git
# Then open the Plugins panel in Codex and install "Grove"
# (or use the CLI install flow)
```

First tool call triggers OAuth in your browser.

### Direct clone (alternative)

If you'd rather skip the marketplace and drop the plugin into Codex's plugins directory directly, clone the repo somewhere and symlink just the plugin subdirectory in:

```bash
git clone https://github.com/grove-tax/grove-mcp.git ~/grove-mcp
ln -s ~/grove-mcp/plugins/grove ~/.codex/plugins/grove
# Restart Codex
```

(The plugin lives in `plugins/grove/` inside the repo — not at the repo root — so the link points at the inner directory. The marketplace install above does the equivalent automatically.)

### Connector-only (no plugin)

If you'd rather skip the plugin layer and just configure the MCP server directly, add this block to `~/.codex/config.toml`:

```toml
[mcp_servers.grove]
type = "http"
url = "https://app.grove.tax/api/mcp"
```

This gives you the MCP tools without the skill (instructions). The skill is the part that teaches Codex how to use Grove well; the tools alone work but Codex will ask more clarifying questions.

## Install in Cursor

Cursor reads the same skill format. Drop `plugins/grove/skills/grove/` into `~/.cursor/skills/` (or your project's `.cursor/skills/`). For the connector itself, add the MCP server in Cursor's settings using the same URL:

```
https://app.grove.tax/api/mcp
```

## Endpoint

Production MCP endpoint: `https://app.grove.tax/api/mcp`

### Pointing at a different endpoint

For non-production testing, edit `plugins/grove/.mcp.json` in your installed copy of the plugin and replace the URL. Do not commit non-production endpoints:

```json
{
  "mcpServers": {
    "grove": {
      "type": "http",
      "url": "https://mcp.example.com/api/mcp"
    }
  }
}
```

Restart Codex (or your client) after changing the endpoint.

## Verify it's working

Once installed, ask your assistant:

> What Grove MCP tools are available?

You should see six tools: `list_returns`, `read_fact_sheet`, `get_checklist`, `request_upload_url`, `upload_return`, and `get_import_status`. If the list comes back empty, the OAuth flow may not have completed — restart the client and try again.

## What's in this repo

```
.
├── README.md                          ← you are here
├── AGENTS.md                          ← Codex CWD-style fallback (used when no plugin/marketplace)
├── .claude-plugin/
│   └── marketplace.json               ← Claude Code marketplace catalog (lists ./plugins/grove)
├── .agents/plugins/
│   └── marketplace.json               ← Codex marketplace catalog (lists ./plugins/grove)
└── plugins/grove/                     ← THE plugin (skill + MCP server + manifests)
    ├── .claude-plugin/plugin.json     ← Claude Code plugin manifest
    ├── .codex-plugin/plugin.json      ← Codex plugin manifest
    ├── .mcp.json                      ← MCP server config (consumed by both)
    └── skills/grove/                  ← the Agent Skill (Claude / Codex / Cursor)
        ├── SKILL.md                   ← main skill body (loads when relevant)
        ├── tools-reference.md         ← per-tool reference
        ├── file-formats/              ← Drake + Lacerte backup format details
        └── workflows.md               ← end-to-end recipes for common preparer flows
```

The repo doubles as a Claude Code marketplace AND a Codex marketplace, each listing one plugin at `./plugins/grove`. That plugin bundles the skill + MCP server + manifests for both ecosystems.

## Troubleshooting

**"Connector failed" / repeated authentication errors.** Your firm may not have MCP access enabled yet. Contact `support@grove.tax` to confirm.

**OAuth sign-in succeeds but tool calls still fail.** You may have signed in with a different account from the one tied to your firm in Grove. Sign out of Grove in your browser, then re-add the connector and sign in with the email you actually use for Grove.

**Browser opens but the consent screen shows an error.** Grove support can verify your firm is configured correctly for MCP — email `support@grove.tax` with the error reference shown on screen.

## License

MIT.
