# ApiVault CLI — Reference

## Configuration Architecture

### Global Configuration (`~/.apivault/`)
Directory: `~/.apivault/` (Windows: `%USERPROFILE%\.apivault\`)

- **`config.json`**: System-wide defaults for `project`, `run.command`, `run.env`, and `vaultKey`.
- **`token.json`**: Device authentication token generated upon `apivault login`.
- Unix file permissions: `0600`.

```json
{
  "project": "proj_default",
  "run": {
    "command": "npm start",
    "env": "Production"
  },
  "vaultKey": "your-custom-vault-key"
}
```

### Local Project Configuration (`.apivault.json`)
Stored in the repository root directory to bind the project context and settings locally.

Canonical file: `.apivault.json` (also parses `.apivaultrc`, `apivault.json`, `.apivault`)

```json
{
  "project": "proj_123abc",
  "run": {
    "command": "npm run dev",
    "env": "Development"
  }
}
```

`.apivault.json` is usually committed so teammates share the project link. Keep `vaultKey` out of it: set the vault key globally, in `APIVAULT_KEY`, or let the CLI prompt.

### Resolution Priority

The full order for each setting is in the Resolution Precedence table in `SKILL.md`: flags, then environment variables, then local config, then global config.

Two exceptions under `APIVAULT_TOKEN`. The **project** is fixed by the token and needs no local config — a contradicting `-p` is refused by the server (403). The **environment** resolves `--env` → the token's pin → local config → global config, so the pin outranks config: a pinned token can only read its own environment, and a fresh CI container has no config to read.

The project value may be a project **id** or its **slug** — the server resolves either. Slugs are unique per owner, so a caller who belongs to two same-slug projects gets the one they own.


## Environment Variables & Flags

| Variable / Flag | Used by | Purpose |
|----------|---------|---------|
| `APIVAULT_TOKEN` | all commands | Service token (`av_live_...`) for headless/CI auth. Outranks `~/.apivault/token.json` |
| `APIVAULT_KEY` / `--vault-key` | reveal, add, update, run, env | Custom vault encryption key |
| `-p, --project <id\|slug>` | all commands | Override the target ApiVault project |
| `APIVAULT_PROJECT` | all commands | Same value as `-p`, from the environment |
| `NO_COLOR` | UI | Disable terminal colors |
| `FORCE_COLOR=0` | UI | Disable terminal colors |

## HTTP API (CLI & Backend)

| Action | Method | Route | Description |
|--------|--------|-------|-------------|
| Login start | POST | `/api/cli/request` | Create one-time CLI pairing request |
| Login poll | POST | `/api/cli/status` | Poll pairing status with requestToken |
| Logout | DELETE | `/api/cli/token` | Revoke CLI API token |
| Whoami | GET | `/api/cli/me` | Return authenticated user metadata |
| List projects | GET | `/api/projects` | List available user workspaces |
| List keys | GET | `/api/keys` | List keys with masked preview |
| Page keys | GET | `/api/keys?limit=<1-500>&cursor=<id>` | One page of keys; the next cursor is in the `X-Next-Cursor` header, absent on the last page |
| Filter keys | GET | `/api/keys?environment=<env>` | Filter keys by environment (or `&service=`) |
| Add key | POST | `/api/keys` | Create single encrypted key |
| Bulk import | POST | `/api/keys/bulk` | Batch import `.env` keys atomically (up to 500 items per request) |
| Update key | PUT | `/api/keys/:id` | Update metadata or encrypted value |
| Delete key | DELETE | `/api/keys/:id` | Delete key permanently |
| Decrypt | POST | `/api/keys/:id/decrypt` | Decrypt and return raw secret |
| List logs | GET | `/api/projects/:projectId/logs` | Audit & request log entries, paginated |
| Poll logs | GET | `/api/projects/:projectId/logs?since=<ISO>` | Entries newer than a checkpoint (live tail) |
| Log filters | GET | `/api/projects/:projectId/logs/filters` | Distinct endpoints, users, methods, sources, statuses, keys, event types |

**Key routes are rate limited.** `/api/keys*` allows 600 requests a minute per user (web, CLI and MCP share it) and per service token, then answers `429 RATE_LIMITED` with `Retry-After`. `apivault run` and `env export` make one request per key, so parallel CI jobs should use separate tokens.

**Service tokens reach only `/api/keys*` and `/api/cli/me`.** Every other route above — `/api/projects`, the logs routes, the CLI pairing routes — resolves a human user and answers a `av_live_` bearer with `401 Unauthorized`. This is a property of how those routes authenticate, not a per-route denylist, so it holds for any route not listed as service-capable.

Request headers:

```
Authorization: Bearer <apiToken | av_live_serviceToken>
User-Agent: apivault-cli/<version>
X-Project-Id: <project id or slug>   # target workspace; omitted for /api/cli/*
X-Vault-Key: <vault_key>             # when custom encryption enabled
```

A service token carries its own project, so `X-Project-Id` is unnecessary; sending one that names a different project is refused with `403`.

`POST /api/keys/:id/decrypt` carries the project as a `?projectId=` query parameter instead of the header. Both accept an id or a slug.

## Standalone Install Scripts

- **macOS / Linux:** `curl -fsSL https://apivault.tech/install.sh | sh`
- **Windows (PowerShell):** `irm https://apivault.tech/install.ps1 | iex`

Installs standalone Node runtime bundle into `~/.local/share/apivault` and links binary without requiring npm or sudo.

## Login Flow

1. `POST /api/cli/request` → `{ requestId, requestToken, expiresAt }`
2. Browser opens to connect page (`/cli/connect?rid=<requestId>`)
3. User approves or refuses
4. CLI polls `POST /api/cli/status { requestToken }`
5. On approval → `{ apiToken, expiresAt, user }` saved to `token.json`

Minted tokens expire 90 days after approval. `apivault login` prints the expiry date,
`apivault whoami` shows it (warning under 14 days), and `apivault --json whoami` returns it as
`tokenExpiresAt` — `null` for tokens saved before expiry tracking existed.

## apivault run Internals

0. Resolve the environment: `--env` → service token pin (one `GET /api/cli/me`, service-token mode only) → local config → global config
1. Fetch keys: `GET /api/keys?environment=<env>`
2. Decrypt each key (prompt for vault key if needed)
3. Rename `.env`, `.env.local`, `.env.*` → `*.apivault-run-hidden`
4. Build env from decrypted keys; set `__NEXT_PROCESSED_ENV=true` for Next.js
5. Spawn child via `cross-spawn` (no shell — Windows `.cmd` compatible)
6. Restore hidden dotenv files on exit or SIGINT/SIGTERM

Empty environment: warning only; command runs without injected secrets.

## env export Internals

0. Resolve the environment exactly as `run` does (see above)
1. Fetch and decrypt keys for environment
2. **Error** if zero keys (stricter than `run`)
3. Parse existing output file if present
4. Merge by default; `--force` replaces entirely
5. Write header comment; `0600` permissions on Unix

## apivault logs Internals

Query parameters map one-to-one onto the flags: `page`, `limit` (max 1000), `status`, `source`,
`method`, `user`, `endpoint`, `eventType`, `apiKey`/`key`, `search`, `date`, `endDate` +
`isRange=true`, and `since`. Repeatable parameters (`status`, `source`, `method`, …) match any of
the supplied values.

- `status` takes `success` (`< 400`), `error` (`>= 400`), or an explicit code such as `404`.
  An unrecognized value returns `400`, as does a malformed `date` / `endDate` / `since`.
- `search` matches the entry's own id (prefix included), endpoint, method, source, IP, event
  type, and the actor's name or email — which is how `logs get <id>` resolves a single entry.
- `since` returns entries with `createdAt` strictly greater than the checkpoint, capped at 50 per
  poll, so `--follow` advances its checkpoint to the newest entry it printed and never repeats one.
- The `:projectId` path segment accepts a project id or slug, like `X-Project-Id`.

`--follow` polls every 2s. Transient failures are retried at the next tick; `400`, `401`, `403`
and `404` stop the stream and surface the error, so an expired token cannot look like idle traffic.

## keys add (Non-Interactive)

```bash
apivault keys add \
  --name "Stripe Production" \
  --service stripe \
  --environment Production \
  --key "sk_live_..." \
  --notes "Main account" \
  --vault-key "your-vault-key"
```

On `keys add`, `--key` is the **API secret value**; `--vault-key` is the vault key.

## CLI Error Codes

| Code / Status | Meaning |
|---------------|---------|
| HTTP 401 | Not signed in, or this device's token expired or was revoked |
| HTTP 0 | Network / connectivity failure |
| `DUPLICATE_KEY` / HTTP 409 | A key with this name already exists in this environment |
| `VAULT_KEY_REQUIRED` | Custom encryption; vault key needed |
| `INVALID_VAULT_KEY` | Wrong vault key |
| `VAULT_KEY_RATE_LIMITED` / HTTP 429 | Too many wrong vault keys; the CLI reports how long to wait |
| `RATE_LIMITED` / HTTP 429 | More than 600 key requests a minute from this user or service token; the CLI reports how long to wait |
| `TOO_MANY_ITEMS` / HTTP 400 | Bulk import over 500 items; nothing was saved, split into batches |
| `INVALID_LIMIT` / `INVALID_CURSOR` / HTTP 400 | `limit` outside 1–500, or a `cursor` that isn't a key in the listing |
| `INSUFFICIENT_SCOPE` / HTTP 403 | Service token lacks the scope for this call |
| `ENVIRONMENT_MISMATCH` / HTTP 403 | Requested environment contradicts the token's pin |
| `SERVICE_TOKEN_EXPIRED` / `SERVICE_TOKEN_REVOKED` | Token is past its expiry, or was revoked |
| `SERVICE_TOKEN_DENIED_IP` / HTTP 403 | Caller's address is outside the token's IP allowlist |

Failed commands are audited alongside successful ones: any of the above that reaches a resolved
project is written to that project's Logs page with its status and error code, attributed to the
signed-in user and the `cli` source. HTTP 401 is the exception — a request rejected before a
project is resolved has no project to be filed against.

A refused **service token** is the one case that escapes that exception. Because the token names
its project even when it is denied, expiry, revocation, IP and scope denials are all filed against
that project with the `service` source and the token's name, so a pipeline's failed attempts are
visible on the Logs page rather than vanishing.

## For CLI Maintainers

The sections below are for people working on the CLI itself, not for using it.

### Server URL

Compiled into the CLI at build time (`src/config.ts` → `API_BASE_URL`, default `https://apivault.tech`). Change and rebuild for local or self-hosted development; it is not configurable at runtime.

### Install From Source

```bash
git clone https://github.com/ApiVault-tech/apivault.git
cd apivault
npm install && npm run build
npm install -g .
```

Development without a global install: `npm run dev -- <command>`

### Source Layout (apivault-cli)

```
apivault-cli/
├── src/
│   ├── index.ts              # Commander entry
│   ├── config.ts             # API_BASE_URL, persistence, APIVAULT_TOKEN mode
│   ├── connect.ts            # login, logout, whoami
│   ├── service-identity.ts   # service token identity + environment resolution
│   ├── http.ts               # ApiClient, ApiError
│   ├── run-env.ts            # dotenv hide/restore
│   ├── env-file.ts           # parse/merge .env
│   ├── commands/
│   │   ├── keys.ts
│   │   ├── run.ts
│   │   ├── config.ts
│   │   ├── env.ts
│   │   ├── projects.ts       # projects list / use / current
│   │   └── logs.ts           # logs list / tail / get
│   └── ui/format.ts          # colors, tables, JSON
├── dist/cli.js
└── package.json              # bin: apivault
```

### Build Scripts

| Script | Command | Purpose |
|--------|---------|---------|
| `dev` | `tsx src/index.ts` | Run without build |
| `build` | `tsup` | Bundle to `dist/cli.js` |
| `typecheck` | `tsc --noEmit` | TypeScript validation |

Requirements: Node.js >= 18.17
