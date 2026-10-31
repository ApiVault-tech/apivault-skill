---
name: apivault
description: >-
  Integrate with ApiVault for encrypted API key management. Covers the remote
  MCP server (OAuth, scoped tools for list/get/reveal/add/update/delete keys)
  and the apivault CLI (browser login, keys CRUD, secret injection via run,
  dotenv export). Use when the user mentions ApiVault, apivault, apivault-cli,
  vault secrets, MCP server for API keys, apivault run, secret injection,
  Cursor/Claude MCP integration, or managing encrypted credentials for AI
  agents and developers. Prefer MCP inside IDE agents; prefer CLI for terminal
  workflows and process injection.
---

# ApiVault

ApiVault is a secure API key vault with two integration surfaces:

| Surface | Best for | Auth |
|---------|----------|------|
| **MCP server** | AI agents in Cursor, Claude Desktop, and other MCP clients | OAuth (browser approval, scoped tokens) |
| **CLI** (`apivault`) | Terminal workflows, CI scripts, injecting secrets into local processes | Browser connect flow (`apivault login`) |

Production endpoint: `https://apivault.tech`

Website: [apivault.tech](https://apivault.tech) · CLI Repo: [github.com/ApiVault-tech/apivault](https://github.com/ApiVault-tech/apivault)

Both surfaces communicate with ApiVault exclusively over HTTPS. Never import ApiVault server code, touch Prisma, or access the database directly.

---

## Security Principles

Apply these in every workflow:

1. **Least privilege** — request only the MCP scopes needed; use masked `list_keys` / `get_key` before calling `reveal_key`.
2. **No secret leakage** — do not paste revealed values into chat, commits, logs, or issue trackers. Inject secrets via env vars or local files the user controls.
3. **Never commit** - `~/.apivault/`, exported `.env` files, tokens, or vault keys.
4. **Separate credentials** — MCP tokens and CLI tokens are independent; revoking one does not revoke the other.
5. **Custom vault mode** - when the user enables custom encryption, a `vault_key` is required for reveal/write operations and **cannot be recovered** if forgotten.

---

## MCP Server (AI Agents)

Remote MCP with OAuth. Ideal when an agent needs to browse or manage vault keys from inside an IDE.

### Project-Scoped Connections

ApiVault supports multi-project workspaces. When an AI agent initiates an MCP connection and opens the browser for OAuth approval, the user will select a **Target Project**. 
The resulting authorization token is permanently bound to that project. All MCP tool operations (like fetching, revealing, or adding keys) automatically route to the selected workspace, requiring no manual project IDs or CLI flags in your MCP tool payloads.

### Setup

Use the `add-mcp` CLI to configure ApiVault MCP automatically across your tools:

```bash
npx -y add-mcp https://mcp.apivault.tech/mcp -g
```

Or add directly to your client config (Cursor, Claude Desktop, Windsurf):

```json
{
  "mcpServers": {
    "apivault": {
      "url": "https://mcp.apivault.tech/mcp"
    }
  }
}
```

On first use, the client opens a browser for OAuth approval. The user selects the target project and scopes, then approves access.

### Authentication

If tools return an auth error or the server status is `needsAuth`:

1. Follow the MCP client's OAuth flow (reconnect the server or use the client's auth tool if provided).
2. User approves scopes and project in the browser.
3. Retry the intended tool.

Manage and revoke connections in ApiVault → **Settings → MCP Connections**.

### OAuth Scopes

| Scope | Permission |
|-------|------------|
| `keys:read` | List and view masked key metadata |
| `keys:write` | Create, update, and delete keys |
| `keys:reveal` | Decrypt and return raw secret values |

Request the minimum scopes required for the task.

### Tools

Discover schemas with MCP tool inspection before calling. Summary:

| Tool | Scope | Purpose |
|------|-------|---------|
| `list_keys` | `keys:read` | List keys (masked). Optional filters: `environment`, `service` |
| `get_key` | `keys:read` | Single key metadata by `id` (masked) |
| `reveal_key` | `keys:reveal` | Decrypt secret. Args: `id`, optional `vault_key` |
| `add_key` | `keys:write` | Create key. Required: `name`, `key`. Optional: `service`, `environment`, `notes`, `vault_key` |
| `update_key` | `keys:write` | Update metadata or value. Args: `id` + fields to change, optional `vault_key` when changing value |
| `delete_key` | `keys:write` | Permanently delete by `id` |

### MCP Workflow Patterns

**Find a key for an integration task**

1. `list_keys` with `environment` filter (e.g. `"Production"`)
2. `get_key` to confirm the correct `id`
3. Only if the raw secret is required: `reveal_key` — then use it immediately in env/config, not in chat

**Add a new credential**

1. Confirm scope includes `keys:write`
2. `add_key` with `name`, `key`, `service`, `environment`
3. Pass `vault_key` when the vault uses custom encryption

**Handle errors**

| Code | Meaning | Action |
|------|---------|--------|
| `INSUFFICIENT_SCOPE` | Token lacks required scope | Re-authenticate with broader scopes or use a different tool |
| `NOT_FOUND` | Key id invalid | Re-list keys |
| `DUPLICATE_KEY` | Key name already exists in this environment | Choose a unique name, change environment, or update existing key |
| `VAULT_KEY_REQUIRED` | Custom encryption; vault key needed | Pass `vault_key` |
| `INVALID_VAULT_KEY` | Wrong vault key | Ask the user for the correct key — never guess or re-send the same one |
| `VAULT_KEY_RATE_LIMITED` | Too many wrong vault keys (10 per 15 min, per user and source address) | Stop. Wait the delay stated in the error, and tell the user rather than retrying |

**Never brute-force a vault key.** Wrong vault keys are rate limited server-side across every
surface (MCP, CLI, and web). A correct key clears the counter, so revealing many keys in a row is
fine; repeatedly submitting a key the user has not confirmed will lock the vault path for them.

**Failed attempts are recorded.** The project's audit log captures refused and failed requests, not
just successful ones — a wrong vault key, a throttled reveal, or a write refused by a read-only
role each lands on the Logs page with its status and error code, attributed to the acting user and
the client name the agent reports. Guessing is visible to the whole project, so ask the user for
the right key instead of trying another.

For OAuth endpoints, token TTLs, and protocol details, see [references/mcp.md](references/mcp.md).

---

## CLI (`apivault`)

Terminal client for developers.

**Install (recommended):**

```bash
npm install -g apivault
apivault --version
```

**On-demand (no install):** `npx apivault <command>`

**Standalone script installers:**
- macOS / Linux: `curl -fsSL https://apivault.tech/install.sh | sh`
- Windows (PowerShell): `irm https://apivault.tech/install.ps1 | iex`

**From source:** [github.com/ApiVault-tech/apivault](https://github.com/ApiVault-tech/apivault)

```bash
git clone https://github.com/ApiVault-tech/apivault.git
cd apivault
npm install && npm run build
npm install -g .
```

Development without global install: `npm run dev -- <command>`

### Requirements

- Node.js >= 18.17
- Reachable ApiVault server

### First-Time Auth

```bash
apivault login            # browser Approve/Refuse; default timeout 300s
apivault whoami           # verify session
apivault logout           # revoke token
```

Local state:
- **Global**: `~/.apivault/` (`%USERPROFILE%\.apivault\` on Windows) — `token.json`, `config.json`
- **Local**: `.apivault.json` / `.apivaultrc` in project repository root

| File | Purpose |
|------|---------|
| `token.json` | CLI auth token |
| `config.json` | Global defaults: `project`, `run.env`, `run.command`, `vaultKey` |
| `.apivault.json` | Local per-project configuration in repository root |

### Headless Auth — Service Tokens (CI/CD)

`apivault login` needs a browser. For pipelines, containers, and scheduled jobs, use a **service token**: a project-scoped machine credential created in the web app under **project → Service Tokens**. Shown once, prefixed `av_live_`, stored only as a hash.

```bash
export APIVAULT_TOKEN="av_live_..."
apivault whoami            # reports the machine identity, not a person
apivault run -- npm start  # no login, no ~/.apivault/, no --env needed
```

`APIVAULT_TOKEN` outranks a stored device token, so it also works on a machine that is already signed in.

| Command | Service token | Required scope |
|---------|---------------|----------------|
| `keys list`, `keys get` | ✅ | `keys:read` |
| `keys get --reveal` | ✅ | `keys:reveal` |
| `keys add` (flags only), `keys delete -f` | ✅ | `keys:write` |
| `keys update` | ❌ | interactive; no flag equivalent |
| `env export`, `run` | ✅ | `keys:read` + `keys:reveal` |
| `whoami`, `config`, `link` | ✅ | any / local |
| `projects list` | ❌ | account-wide, not project-scoped |
| `logs`, `logs tail`, `logs get` | ❌ | audit surface — read in the web app |
| `login`, `logout` | ❌ | refused while `APIVAULT_TOKEN` is set |

**Environment pinning.** A token may be pinned to one environment. When it is, `run` and `env export` use it automatically — do not add `--env`. An explicit `--env` that contradicts the pin fails with `ENVIRONMENT_MISMATCH` rather than silently reading the wrong secrets. An unpinned token has no default and requires `--env`.

**Writing without a terminal.** `keys add` prompts for anything not passed as a flag, so in CI supply them: `--name` and `--key` are required, `--environment` falls back to the token's pin (required when unpinned), `--service` defaults to `Custom`. A missing flag is reported as an error rather than hanging. `keys delete` requires `-f` (or `--json`) — without a terminal to confirm on it refuses rather than assuming yes. `keys update` prompts for every field with no flag equivalent — to replace a value headlessly, `keys delete -f` then `keys add`. On a custom-vault-key project pass `--vault-key` or set `APIVAULT_KEY`, or any decrypting command stops for lack of a prompt.

**Do not suggest `apivault login` when `APIVAULT_TOKEN` is set** — it refuses by design. Service tokens are revoked from the project's Service Tokens page, never with `apivault logout`.

### Command Reference

Global flags: `--json`, `--timeout <seconds>`, `-p/--project <id|slug>`, `-V/--version`, `-h/--help`

Anywhere a project is named — `-p/--project`, `APIVAULT_PROJECT`, or the `project` config value — pass either the project id or its slug (the name in the project URL, `apivault.tech/<username>/<slug>`). `apivault projects list` prints both.

```bash
# Project Context & Local Directory Bindings
apivault projects list                       # list available workspaces (id + slug)
apivault projects use <id|slug>              # link current directory (.apivault.json)
apivault link <id|slug>                      # shortcut to link directory
apivault unlink                              # remove local project binding
apivault projects current                    # show active project & resolution source

# Keys
apivault keys list
apivault keys get <id> [--reveal] [--vault-key <vault_key>]
apivault keys add [--name --service --environment --key --notes] [--vault-key <vault_key>]
apivault keys update <id>                    # interactive only
apivault keys delete <id> [-f]

# Run — inject secrets into a child process (preferred for local dev)
apivault run --env <env> [--vault-key <vault_key>] -- <command> [args...]
apivault run                                 # uses config defaults

# Config (supports --local / -l and --global / -g)
apivault config list [--local | --global]
apivault config get <key> [--local | --global]
apivault config set <key> [value] [--local | --global]
apivault config delete <key> [--local | --global]
# Valid keys: project, run.command, run.env, vaultKey

# Dotenv files
apivault env export --env <env> [-o path] [--force] [--vault-key <vault_key>]
apivault env restore [-C <project-dir>]

# Audit & request logs (read-only; no secret values are ever logged)
apivault logs                                # recent entries for the active project
apivault logs [-n <limit>] [--page <n>]
apivault logs --status <success|error|CODE>  # repeatable; CODE e.g. 200, 404
apivault logs --source <cli|mcp|web|api|service|integration> --method <GET|POST|PUT|PATCH|DELETE>
apivault logs --event-type <KEY_REVEALED|KEY_ROTATED|...> --user <userId> --key <idOrName>
apivault logs --date <YYYY-MM-DD> [--end-date <YYYY-MM-DD>]
apivault logs --search <query>               # log id, endpoint, method, source, IP, event, actor
apivault logs --follow                       # stream new entries; `apivault logs tail` is an alias
apivault logs get <id>                       # full detail; accepts an id prefix
```

Use `logs` to answer "what happened to this key / project" without revealing any secret: entries
carry endpoint, status, actor, duration, and a details payload, never a key value. With `--json`,
`logs` returns the response object while `--follow` emits one JSON object per line.

### Resolution Precedence

| Setting | Order (first wins) |
|---------|-------------------|
| Project Context (id or slug) | `-p/--project` → `APIVAULT_PROJECT` env → Local `.apivault.json` → Global `config.json` → first user project |
| Environment | `--env` → service token's environment pin → Local `run.env` → Global `run.env` → error |
| Run command | args after `--` → Local `run.command` → Global `run.command` → error |
| Vault key | `--vault-key` → `APIVAULT_KEY` env → Local `vaultKey` → Global `vaultKey` → prompt |


`APIVAULT_KEY` applies to reveal, add, update, run, and env export. On `keys add`, `--key` is the API secret value; `--vault-key` is the vault key.

### CLI Workflow Patterns

**Run app without writing secrets to disk**

```bash
apivault config set run.env Production
apivault config set run.command "npm start"
apivault run
```

`run` temporarily renames local `.env*` files to `*.apivault-run-hidden`. Restores on exit. If restore fails: `apivault env restore`.

**Export for frameworks that require dotenv files**

```bash
apivault env export --env Production
apivault env export --env Staging -o .env.local --force
```

**Scripting**

```bash
apivault --json keys list
apivault --json keys get <id> --reveal
```

For HTTP routes, config schema, and CLI internals, see [references/cli.md](references/cli.md).

---

## CLI vs MCP — Decision Guide

| Task | Use |
|------|-----|
| Agent needs to look up or manage keys during a coding session | **MCP** |
| User wants secrets injected into `npm start` / local dev server | **CLI** `run` |
| CI/CD pipeline, container, or scheduled job | **CLI** with `APIVAULT_TOKEN` service token |
| Shell script on a developer's own machine | **CLI** with `--json` |
| Export `.env` for Docker / Next.js / Vite | **CLI** `env export` |
| User asks to connect ApiVault to an MCP client | **MCP** setup |
| Revoke AI agent access | ApiVault **Settings → MCP Connections** |
| Revoke terminal access | `apivault logout` |
| Revoke a pipeline's access | ApiVault **project → Service Tokens → Revoke** |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| MCP tools unavailable | Not authenticated | Reconnect apivault MCP server; complete OAuth in browser |
| `INSUFFICIENT_SCOPE` | Token missing scope | Re-connect with required scopes |
| CLI HTTP 401 | Not logged in, or the token expired (CLI tokens last 90 days) | `apivault login`; check the expiry with `apivault --json whoami` |
| `APIVAULT_TOKEN was not recognised` | Service token mistyped, deleted, or from another instance | Re-copy the full `av_live_...` value; do **not** suggest `apivault login` |
| `ENVIRONMENT_MISMATCH` | `--env` contradicts the token's environment pin | Drop `--env` and let the pin apply, or use a token pinned to that environment |
| `INSUFFICIENT_SCOPE` on `keys add/update/delete` | Service token lacks `keys:write` | Issue a token with the scope; scopes cannot be widened after creation |
| `not available to service tokens` | `projects list` / `logs` under `APIVAULT_TOKEN` | Use the web app, or run as a signed-in user |
| `VAULT_KEY_REQUIRED` | Custom encryption vault | Provide `vault_key` / `--vault-key` / `APIVAULT_KEY` |
| HTTP 429 / `VAULT_KEY_RATE_LIMITED` | Too many wrong vault keys | Wait for the stated delay; do not retry the same key |
| Secrets not loading in `run` | Wrong environment or no keys | Check `list_keys` / `keys list` for environment name |
| `.env` files missing after `run` | Interrupted restore | `apivault env restore` |
| PowerShell strips `--` | Shell parsing | CLI falls back to remaining args; quote command if needed |
| Local dev against self-hosted ApiVault | Compiled server URL | Change `API_BASE_URL` in CLI `src/config.ts`, rebuild |

---

## Development

**CLI repo** — key scripts: `npm run dev`, `npm run build`, `npm run typecheck`

**ApiVault server** — MCP implementation: `src/mcp/server.ts`, route: `src/app/mcp/route.ts`

Local MCP base URL: set `MCP_BASE_URL` env on the ApiVault server.
