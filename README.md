# ApiVault Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![ApiVault Web](https://img.shields.io/badge/ApiVault-Web%20App-6366f1.svg)](https://apivault.tech)
[![CLI Version](https://img.shields.io/badge/CLI-v1.7.3-10b981.svg)](https://github.com/ApiVault-tech/apivault)
[![MCP Protocol](https://img.shields.io/badge/MCP-OAuth%202.1-8b5cf6.svg)](https://modelcontextprotocol.io)

Official AI agent skill for **[ApiVault](https://apivault.tech)** — the encrypted API key management vault.

This skill equips AI coding assistants, IDE extensions, and autonomous agents with complete knowledge and workflows for:
1. **Remote Model Context Protocol (MCP) Server** — Scoped, revocable OAuth 2.1 integration to query, manage, and reveal API keys securely.
2. **ApiVault CLI (`apivault`)** — Terminal authentication, secret injection into child processes without disk persistence (`apivault run`), and dotenv export.

---

## Supported Platforms & Clients

This skill works across any modern AI coding assistant, IDE, or agent framework:

| Integration Surface | Supported Clients / Platforms | How It Works |
| :--- | :--- | :--- |
| **Remote MCP Server** | Cursor, Claude Desktop, VS Code (Cline / Roo Code), Windsurf | Direct tool execution via OAuth 2.1 (`list_keys`, `reveal_key`, `add_key`, etc.) |
| **Workspace / Project Skill** | Cursor, Windsurf, Claude Code, Antigravity, GitHub Copilot | Context-aware runbooks and instructions in `.agents/skills/` |
| **Global Agent Skill** | Claude Code (`~/.claude/`), Antigravity (`~/.gemini/config/`), CLI tools | Machine-wide skill discovery across all projects |

---

## Installation & Setup

### Option A: Connect as an MCP Server (Cursor, Claude Desktop, VS Code)

Add the ApiVault remote MCP endpoint to your client configuration (e.g. `mcp.json` or client settings):

```json
{
  "mcpServers": {
    "apivault": {
      "url": "https://mcp.apivault.tech/mcp"
    }
  }
}
```

*Pre-configured templates are available in the [examples/](examples/) directory (`cursor-mcp.json`, `claude-desktop-mcp.json`, `vscode-mcp.json`).*

On first connection, the client will open a browser window for OAuth 2.1 authorization. You will select a target project workspace and approve the requested scopes.

---

### Option B: Add to a Project Workspace

Clone the skill into your project's agent skills directory to give any workspace agent access:

```bash
mkdir -p .agents/skills/apivault
git clone https://github.com/ApiVault-tech/apivault-skill.git .agents/skills/apivault
```

---

### Option C: Global Machine-Wide Installation

Install into your preferred global agent configuration directory:

```bash
# Claude Code
mkdir -p ~/.claude/skills
git clone https://github.com/ApiVault-tech/apivault-skill.git ~/.claude/skills/apivault

# Antigravity / Gemini CLI
mkdir -p ~/.gemini/config/skills
git clone https://github.com/ApiVault-tech/apivault-skill.git ~/.gemini/config/skills/apivault
```

---

### 🤖 Quick Agent Prompt (Copy & Paste)

You can ask any AI assistant to install and use this skill by pasting:

```text
Please install the ApiVault skill for this workspace by cloning https://github.com/ApiVault-tech/apivault-skill.git into .agents/skills/apivault. Read .agents/skills/apivault/SKILL.md to learn how to manage ApiVault keys and connect to the MCP server.
```

---

## Repository Structure

```text
apivault-skill/
├── SKILL.md                  # Main instruction file with YAML frontmatter & progressive disclosure
├── README.md                 # Documentation and setup guide
├── LICENSE                   # MIT License
├── references/               # In-depth technical documentation (loaded on-demand)
│   ├── mcp.md                # MCP endpoint, OAuth 2.1 PKCE, tool schemas & error codes
│   └── cli.md                # CLI commands, ~/.apivault/ schemas & API route matrix
└── examples/                 # Ready-to-copy client configs and scripts
    ├── cursor-mcp.json       # MCP configuration for Cursor
    ├── claude-desktop-mcp.json # MCP configuration for Claude Desktop
    ├── vscode-mcp.json       # MCP configuration for VS Code / Cline
    ├── config.json           # Sample ~/.apivault/config.json
    ├── cli-workflows.sh      # Sample Bash automation
    └── cli-workflows.ps1     # Sample PowerShell automation
```

---

## Quick Reference: MCP vs CLI

| Scenario | Recommended Surface | Action / Tool |
| :--- | :--- | :--- |
| Agent needs to inspect or manage credentials during coding | **MCP Server** | `list_keys`, `get_key`, `reveal_key` |
| Developer wants secrets injected into `npm start` / dev server | **CLI** | `apivault run -- npm start` |
| CI/CD pipeline, container, or scheduled job | **CLI (service token)** | `APIVAULT_TOKEN=av_live_... apivault run -- npm start` |
| Shell script on a developer's own machine | **CLI (JSON)** | `apivault --json keys list` |
| Exporting `.env` for Docker Compose / Next.js | **CLI** | `apivault env export --env Production` |
| Audit and revoke agent connections | **Web App** | Settings → MCP Connections |
| Terminate CLI token on a machine | **CLI** | `apivault logout` |
| Revoke a pipeline's credential | **Web App** | Project → Service Tokens → Revoke |

---

## Security Principles

- **Least Privilege**: Request only the required MCP scopes (`keys:read`, `keys:write`, `keys:reveal`). The same three scopes apply to service tokens, which can additionally be pinned to one environment and restricted to an IP allowlist.
- **Machines Are Not People**: A service token (`av_live_...`) belongs to a project, not a user. It can reach keys and nothing else — never project administration, the audit log, or another project — and it cannot mint further credentials.
- **No Secret Leakage**: Raw secret values decrypted via `reveal_key` or `apivault keys get --reveal` must never be echoed into chat logs, commits, or issue trackers.
- **Process Memory Isolation**: `apivault run` injects decrypted environment variables directly into process memory and temporarily moves `.env` files aside (`*.apivault-run-hidden`).
- **Custom Vault Key**: In custom encryption mode, the `vault_key` is sent with each request that needs it. ApiVault does not store it and keeps only a hash to verify it.
- **Everything Is Audited**: Refused and failed requests are logged next to successful ones — a wrong vault key, a throttled reveal, or a write blocked by a read-only role appears on the project's Logs page with its status, error code and the agent's reported client name. Assume every attempt is visible to the project.

---

## Official Links

- **Web App**: [https://apivault.tech](https://apivault.tech)
- **Documentation**: [https://apivault.tech/docs](https://apivault.tech/docs)
- **CLI Repository**: [github.com/ApiVault-tech/apivault](https://github.com/ApiVault-tech/apivault)
- **Issues & Feedback**: [github.com/ApiVault-tech/apivault/issues](https://github.com/ApiVault-tech/apivault/issues)

---

## License

[MIT](LICENSE) © 2026 TLB STATION
