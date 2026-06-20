# SolidWorks Design Assistant — Claude Code Plugin

A Claude Code plugin that brings AI-assisted SolidWorks CAD design directly into your workflow, powered by a curated knowledge base of SolidWorks documentation, macros, and design patterns.

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

## Setup

### 1. Configure the knowledge base connection

Copy `.mcp.json` and fill in your server details:

```json
{
  "mcpServers": {
    "solidworks-kb": {
      "url": "http://<your-kb-host>/mcp",
      "headers": {
        "Authorization": "Bearer <your-api-key>"
      }
    }
  }
}
```

> The knowledge base server hosts SolidWorks documentation, feature references, and macro templates used by the plugin's skills.

### 2. Install into Claude Code

```bash
# From the Claude Code CLI
claude plugin install <path-or-repo-url>
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
