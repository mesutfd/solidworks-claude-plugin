# SolidWorks Design Assistant — Claude Code Plugin

A Claude Code plugin that brings AI-assisted SolidWorks CAD design directly into your workflow, powered by a curated knowledge base of SolidWorks documentation, macros, and design patterns.

## Installation

> **Prerequisites:** [Claude Code](https://docs.claude.com/en/docs/claude-code) v1.0.0 or newer.

### Step 1 — Add the marketplace

In an open Claude Code session, register this repo as a plugin marketplace:

```
/plugin marketplace add mesutfd/solidworks-claude-plugin
```

Claude Code reads `.claude-plugin/marketplace.json` from the repo and registers a marketplace named `solidworks-claude-plugin`.

### Step 2 — Install the plugin

```
/plugin install solidworks-claude-plugin@solidworks-claude-plugin
```

The format is `<plugin-name>@<marketplace-name>` — both happen to be `solidworks-claude-plugin` here. Confirm the install when prompted, then restart Claude Code (or run `/plugin` and reload) so the skills, agents, and hooks activate.

> **Tip:** You can also browse and install interactively by running `/plugin` and selecting the marketplace from the menu.

### Step 3 — Configure the knowledge base connection (optional)

The plugin talks to a remote SolidWorks knowledge base over MCP. The knowledge base is **public — no API key or login required**, and it defaults to `https://sw-plugin.ideep.org`, so the plugin works out of the box.

You only need this step if you run your own server. Edit (or create) `.mcp.json` and point it at your host:

```json
{
  "mcpServers": {
    "solidworks-kb": {
      "type": "http",
      "url": "https://sw-plugin.ideep.org/mcp"
    }
  }
}
```

Swap the `url` for your own server if you host the knowledge base yourself.

### Step 4 — Verify

Run `/plugin` and confirm **SolidWorks Design Assistant** is listed and enabled. Start a SolidWorks task — the `pre-start` skill loads conventions and design rules before the first action, which confirms the plugin and knowledge base are wired up.

### Updating & removing

```
# pull the latest plugin version
/plugin marketplace update solidworks-claude-plugin

# uninstall
/plugin uninstall solidworks-claude-plugin@solidworks-claude-plugin
```

## Features (planned)

- Slash commands for common SolidWorks tasks (sketches, features, assemblies, drawings)
- Skills for querying the CAD knowledge base
- Subagents for multi-step design workflows
- MCP connector to the knowledge base API

## Directory Structure

```
solidworks-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json        # plugin metadata
│   └── marketplace.json   # marketplace catalog
├── commands/              # slash commands
├── agents/                # subagents
├── skills/                # skills
├── hooks/                 # lifecycle hooks
├── .mcp.json              # MCP server config (knowledge base API)
└── README.md
```

## Knowledge Base

The plugin connects to a remote knowledge base server that indexes:

- SolidWorks feature documentation
- VBA/API macro templates
- Assembly and drawing best practices
- Design pattern references

API connector details will live in `skills/` once implemented.

## License

MIT
