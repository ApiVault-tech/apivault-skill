---
name: apivault
description: >-
  Work with ApiVault, the encrypted API key vault, through its remote MCP
  server (OAuth; list, get, reveal, add, update and delete keys) and the
  apivault CLI (browser login, keys commands, `apivault run` secret injection,
  .env export, service tokens for CI, audit logs). Use whenever the user
  mentions ApiVault, apivault, apivault-cli, APIVAULT_TOKEN, APIVAULT_KEY or
  av_live_ tokens, or wants API keys from their vault loaded into an app, a
  .env file, a CI pipeline or an MCP client such as Cursor or Claude, even
  without naming ApiVault when the project has a .apivault.json file. Not for
  HashiCorp Vault, AWS Secrets Manager or other secret stores. Prefer MCP
  inside IDE agents; prefer the CLI for terminal, CI and process injection.
---

# ApiVault

ApiVault is a secure API key vault with two integration surfaces:

| Surface | Best for | Auth |
|---------|----------|------|
| **MCP server** | AI agents in Cursor, Claude Desktop, and other MCP clients | OAuth (browser approval, scoped tokens) |
| **CLI** (`apivault`) | Terminal workflows, CI scripts, injecting secrets into local processes | Browser connect flow (`apivault login`), or a service token in CI |

Production endpoint: `https://apivault.tech`

Website: [apivault.tech](https://apivault.tech) · CLI Repo: [github.com/ApiVault-tech/apivault](https://github.com/ApiVault-tech/apivault)

Both surfaces communicate with ApiVault exclusively over HTTPS. Never import ApiVault server code, touch Prisma, or access the database directly.

---

## Security Principles

Apply these in every workflow:

1. **Least privilege** — request only the MCP scopes needed; use masked `list_keys` / `get_key` before calling `reveal_key`.
2. **Keep secrets out of your own context.** Anything you reveal — a `reveal_key` result, or `keys get --reveal` run in your shell — lands in the conversation transcript. When the goal is to give an app its secrets, have the CLI load them straight into the process (`apivault run`) or a file the user controls (`apivault env export`), so the values never pass through you. Reveal a value only when the task needs the raw text, and then never repeat it in chat, commits, logs, or issue trackers.
3. **Never commit** `~/.apivault/`, exported `.env` files, tokens, or vault keys. `.apivault.json` sits in the repository root and is usually committed so teammates share the project link, so it must never hold a `vaultKey` — keep the vault key in `APIVAULT_KEY`, the global config, or the prompt.
4. **Confirm before destroying data.** `delete_key` and `keys delete` are permanent, and changing a value with `update_key` replaces the old one. Confirm the exact key and environment with the user first, and never add `-f` to `keys delete` unless the user asked for a non-interactive delete.
5. **Separate credentials** — MCP tokens and CLI tokens are independent; revoking one does not revoke the other.
6. **Custom vault mode** — when the user enables custom encryption, a vault key is required for reveal/write operations and **cannot be recovered** if forgotten. Never guess it, and don't ask the user to paste it into the chat: over MCP it belongs in the connection's `X-Vault-Key` header (see [Custom vault keys over MCP](#custom-vault-keys-over-mcp)); for the CLI, in `APIVAULT_KEY` or the prompt.

---

## MCP Server (AI Agents)

Remote MCP with OAuth. Ideal when an agent needs to browse or manage vault keys from inside an IDE.

### Project-Scoped Connections

ApiVault supports multi-project workspaces. When an AI agent initiates an MCP connection and opens the browser for OAuth approval, the user selects a **Target Project**.
The resulting authorization token is permanently bound to that project. All MCP tool operations route to that workspace, so tool payloads need no project ids. To work in another project, the user connects again and picks it.

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
3. Retry the intended tool once. Don't loop on auth failures without the user.

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
| `reveal_key` | `keys:reveal` | Decrypt secret. Args: `id` |
| `add_key` | `keys:write` | Create key. Required: `name`, `key`. Optional: `service`, `environment`, `notes` |
| `update_key` | `keys:write` | Update metadata or value. Args: `id` + fields to change |

`reveal_key`, `add_key` and `update_key` also accept an optional `vault_key`. Leave it empty: it
exists only for clients that can't send the `X-Vault-Key` header, and it is ignored when the
header is set.

### Custom vault keys over MCP

A project with a custom vault key needs it to reveal, add or change values. The user sets it once
as an `X-Vault-Key` header on the MCP server entry in their client, read from an environment
variable or a prompt, and the server uses it for every tool. The key never passes through the
chat, so you never see it.

```jsonc
// Cursor (.cursor/mcp.json)
{ "mcpServers": { "apivault": { "url": "https://mcp.apivault.tech/mcp", "headers": { "X-Vault-Key": "${env:APIVAULT_KEY}" } } } }

// Claude Code (.mcp.json)
{ "mcpServers": { "apivault": { "type": "http", "url": "https://mcp.apivault.tech/mcp", "headers": { "X-Vault-Key": "${APIVAULT_KEY}" } } } }
```

VS Code can prompt for it instead: a `promptString` input with `"password": true`, referenced
as `${input:<id>}` in `headers`. When a tool returns `VAULT_KEY_REQUIRED`, help the user add the
header rather than asking for the key. Only if their client can't add headers to a remote server
(such as Claude's Connectors UI) and they explicitly choose to give the key in the chat, pass it as
`vault_key`, and tell them it will stay in the conversation history.
| `delete_key` | `keys:write` | Permanently delete by `id` |

### MCP Workflow Patterns

**Find a key for an integration task**

1. `list_keys` with an `environment` filter (e.g. `"Production"`) — never assume key ids or environment names
2. `get_key` to confirm the correct `id`
3. Only if the raw secret is required: `reveal_key`, then write it straight into the user's env file or config

**Add a new credential**

1. Confirm scope includes `keys:write`
2. Confirm the environment and service with the user when they're ambiguous
3. `add_key` with `name`, `key`, `service`, `environment`
4. On a custom-vault-key project the vault key comes from the connection header; you don't pass it

**Rotate or remove a credential**

1. `list_keys` / `get_key` to find the exact key, and confirm it with the user
2. Rotate with `update_key` and the new `key` value; remove with `delete_key` only after the user agrees

### Errors

These codes come back from both the MCP tools and the CLI.

| Code | Meaning | Action |
|------|---------|--------|
| `INSUFFICIENT_SCOPE` | Token lacks the required scope | MCP: reconnect with the scope. Service token: issue a new token, since scopes can't be widened |
| `NOT_FOUND` | Key id invalid | Re-list keys |
| `DUPLICATE_KEY` | Key name already exists in this environment | Choose a unique name, change environment, or update the existing key (with the user's OK) |
| `VAULT_KEY_REQUIRED` | Custom encryption; vault key needed | MCP: have the user add the `X-Vault-Key` header. CLI: have them set `APIVAULT_KEY` or answer the prompt |
| `INVALID_VAULT_KEY` | Wrong vault key | Have the user correct it (the header, or `APIVAULT_KEY`) — never guess or re-send the same one |
| `VAULT_KEY_RATE_LIMITED` | Too many wrong vault keys (10 per 15 min, per user and source address) | Stop. Wait the delay stated in the error, and tell the user rather than retrying |
| `RATE_LIMITED` | More than 600 key requests in a minute; a user's web, CLI and MCP calls share one budget, each service token has its own | Wait the stated delay; don't loop over keys faster than that, and give parallel CI jobs separate service tokens |
| `ENVIRONMENT_MISMATCH` | `--env` contradicts the service token's environment pin | Drop `--env` and let the pin apply, or use a token pinned to that environment |

**Never brute-force a vault key.** Wrong vault keys are rate limited server-side across every
surface (MCP, CLI, and web). A correct key clears the counter, so revealing many keys in a row is
fine; repeatedly submitting a key the user has not confirmed will lock the vault path for them.

**Failed attempts are recorded.** The project's audit log captures refused and failed requests, not
just successful ones — a wrong vault key, a throttled reveal, or a write refused by a read-only
role each lands on the Logs page with its status and error code, attributed to the acting user and
the client name the agent reports. Guessing is visible to the whole project, so ask the user for
the right key instead of trying another.

For OAuth endpoints, token TTLs, and full tool schemas, see [references/mcp.md](references/mcp.md).

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

Requires Node.js >= 18.17 (the standalone installers bundle their own).

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
| `token.json` | CLI auth token — never commit or share |
| `config.json` | Global defaults: `project`, `run.env`, `run.command`, `vaultKey` |
| `.apivault.json` | Local per-project settings (`project`, `run.env`, `run.command`) — usually committed, so no `vaultKey` |

### Headless Auth — Service Tokens (CI/CD)

`apivault login` needs a browser. For pipelines, containers, and scheduled jobs, use a **service token**: a project-scoped machine credential created in the web app under **project → Service Tokens**. Shown once, prefixed `av_live_`, stored only as a hash. Put it in the CI system's secret store, never in the repository.

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

**Writing without a terminal.** `keys add` prompts for anything not passed as a flag, so in CI supply them: `--name` and `--key` are required, `--environment` falls back to the token's pin (required when unpinned), `--service` defaults to `Custom`. A missing flag is reported as an error rather than hanging. `keys delete` requires `-f` (or `--json`) — without a terminal to confirm on it refuses rather than assuming yes. `keys update` prompts for every field with no flag equivalent — to replace a value headlessly, `keys delete -f` then `keys add`. On a custom-vault-key project set `APIVAULT_KEY` from the CI secret store, or any decrypting command stops for lack of a prompt.

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
apivault keys delete <id> [-f]               # permanent; confirm with the user first

# Run — inject secrets into a child process (preferred for local dev)
apivault run --env <env> [--vault-key <vault_key>] -- <command> [args...]
apivault run                                 # uses config defaults

# Config (supports --local / -l and --global / -g)
apivault config list [--local | --global]
apivault config get <key> [--local | --global]
apivault config set <key> [value] [--local | --global]
apivault config delete <key> [--local | --global]
# Valid keys: project, run.command, run.env, vaultKey (vaultKey: global only, never --local)

# Dotenv files
apivault env export --env <env> [-o path] [--force] [--vault-key <vault_key>]
apivault env restore [-C <project-dir>]

# Audit & request logs (read-only; no secret values are ever logged)
apivault logs                                # recent entries for the active project
apivault logs [-n <limit>] [--page <n>]
apivault logs --status <success|error|CODE>  # repeatable; CODE e.g. 200, 404
apivault logs --source <cli|mcp|web|api|service|integration> --method <GET|POST|PUT|DELETE>
apivault logs --event-type <KEY_REVEALED|KEY_ROTATED|...> --user <userId> --key <idOrName>
apivault logs --date <YYYY-MM-DD> [--end-date <YYYY-MM-DD>]
apivault logs --search <query>               # log id, endpoint, method, source, IP, event, actor
apivault logs --follow                       # stream new entries; `apivault logs tail` is an alias
apivault logs get <id>                       # full detail; accepts an id prefix
```

`--vault-key` on the command line is saved in shell history. When you run commands for the user,
prefer `APIVAULT_KEY` set in their environment, or let the CLI prompt them.

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

The CLI still reads a local `vaultKey` if one exists, but never set one: `.apivault.json` is usually committed. If you find a `vaultKey` in a repository's `.apivault.json`, tell the user — the key is exposed and should be moved out.

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

Check that the output file is covered by `.gitignore` before exporting.

**Scripting**

```bash
apivault --json keys list                    # masked metadata — safe to read
apivault --json keys get <id> --reveal       # raw secret — for the user's scripts, not your shell
```

Write `--reveal` into scripts the user runs (piping the value where it is needed), rather than
running it yourself: its output would land in your context (see Security Principle 2).

For config files, HTTP routes, and CLI internals, see [references/cli.md](references/cli.md).

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

For error codes, see [Errors](#errors) above.

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| MCP tools unavailable | Not authenticated | Reconnect apivault MCP server; complete OAuth in browser |
| CLI HTTP 401 | Not logged in, or the token expired (CLI tokens last 90 days) | `apivault login`; check the expiry with `apivault --json whoami` |
| `APIVAULT_TOKEN was not recognised` | Service token mistyped, deleted, or from another instance | Re-copy the full `av_live_...` value; do **not** suggest `apivault login` |
| `not available to service tokens` | `projects list` / `logs` under `APIVAULT_TOKEN` | Use the web app, or run as a signed-in user |
| Secrets not loading in `run` | Wrong environment or no keys | Check `list_keys` / `keys list` for the environment name |
| `.env` files missing after `run` | Interrupted restore | `apivault env restore` |
| PowerShell strips `--` | Shell parsing | CLI falls back to remaining args; quote command if needed |
