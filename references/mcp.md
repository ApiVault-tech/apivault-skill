# ApiVault MCP Server — Reference

## Endpoint

```
https://mcp.apivault.tech/mcp
```

Self-hosted: `{MCP_BASE_URL}/mcp` (defaults to production URL).

## Protocol

- Transport: Streamable HTTP (MCP over HTTP)
- Auth: OAuth 2.1 with PKCE (S256)
- Dynamic client registration supported

## OAuth Metadata

| Document | Path |
|----------|------|
| Protected Resource Metadata | `/.well-known/oauth-protected-resource` |
| Authorization Server Metadata | `/.well-known/oauth-authorization-server` |
| Server Card | `/.well-known/mcp/server-card` or `/mcp/server-card` |

Key endpoints:

| Endpoint | Path |
|----------|------|
| Authorize | `/oauth/authorize` |
| Token | `/oauth/token` |
| Register client | `/oauth/register` |
| Revoke | `/oauth/revoke` |
| Consent UI | `/mcp/authorize` |
| Server Card | `/mcp/server-card` |

Supported grants: `authorization_code`, `refresh_token`
Response type: `code`
PKCE: `S256`

## Scopes

| Scope | Label | Grants |
|-------|-------|--------|
| `keys:read` | View API keys (masked) | `list_keys`, `get_key` |
| `keys:write` | Add, update, and delete API keys | `add_key`, `update_key`, `delete_key` |
| `keys:reveal` | Reveal decrypted secret values | `reveal_key` |

If the client omits scope in the auth request, all three scopes are offered by default.

## Project-Scoped Connections

During the browser-based authorization flow (`/mcp/authorize`), the user selects a **Target Project**. The issued OAuth access token is permanently bound to this `projectId`.

All tool executions (reading, writing, and revealing keys) automatically route to the selected project workspace without requiring the `projectId` to be passed in the JSON-RPC request.


## Token Lifetimes

| Token | TTL |
|-------|-----|
| Access token | 1 hour |
| Refresh token | 30 days |
| Authorization code | 5 minutes |

## Tool Schemas

### list_keys

List API keys (values masked).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `environment` | string | No | Filter by environment name |
| `service` | string | No | Filter by service name |

Returns: `{ keys: [...] }`

### get_key

Get metadata for a single key (masked).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | Key id |

Returns: `{ key: {...} }` or `NOT_FOUND`.

### reveal_key

Decrypt and return the raw secret.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | Key id |
| `vault_key` | string | No | Custom vault key (required when vault uses custom encryption) |

Returns decrypted key object including raw value.

**Scope required:** `keys:reveal`

### add_key

Create a new API key.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Display name |
| `key` | string | Yes | Raw secret value |
| `service` | string | No | Service label |
| `environment` | string | No | Environment label |
| `notes` | string | No | Optional notes |
| `vault_key` | string | No | Vault key when custom encryption is enabled |

Returns: `{ key: {...} }` (created record, masked).

### update_key

Update an existing key's metadata or value.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | Key id |
| `name` | string | No | New name |
| `service` | string | No | New service |
| `environment` | string | No | New environment |
| `notes` | string | No | New notes |
| `key` | string | No | New raw secret value |
| `vault_key` | string | No | Required when changing value on custom-encryption vault |

Returns: `{ key: {...} }` (updated record, masked).

### delete_key

Permanently delete a key.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | Key id |

Returns confirmation object.

## Error Codes

| Code | Meaning |
|------|---------|
| `INSUFFICIENT_SCOPE` | Token lacks the scope required for this tool |
| `NOT_FOUND` | Key id does not exist |
| `DUPLICATE_KEY` | Key with this name already exists in the environment |
| `VAULT_KEY_REQUIRED` | Custom encryption; `vault_key` must be provided |
| `INVALID_VAULT_KEY` | Provided vault key is incorrect |
| `VAULT_KEY_RATE_LIMITED` | Too many incorrect vault keys — 10 per 15 minutes, per user and source address (HTTP 429 with `Retry-After`) |
| `RATE_LIMITED` | More than 600 key requests in a minute; shared with the user's web and CLI use (HTTP 429 with `Retry-After`) |
| `VALIDATION` | Required fields missing or invalid |
| `DECRYPT_FAILED` | Secret decryption failed |
| `INTERNAL` | Unexpected server error |

Errors return `isError: true` with a text message in tool content.

`reveal_key`, `add_key` and `update_key` verify a supplied `vault_key` against a stored hash, so
wrong keys are throttled. A correct key clears the counter. On `VAULT_KEY_RATE_LIMITED` the tool
message states the wait in seconds — surface it to the user instead of retrying.

Failed tool calls are audited, not only successful ones. A refused or errored call is written to
the project's Logs page with its HTTP status, the error code above, the acting user, the reported
client name and the tool that was invoked — filed under the action attempted, so a denied
`reveal_key` appears as a `KEY_REVEALED` entry with a 4xx status. Calls rejected before a project
can be resolved (an invalid token, or a token with no accessible project) are not logged, because
entries are stored per project.

## Server Instructions (embedded)

The MCP server advertises this instruction string to clients:

> ApiVault MCP server. Use list_keys to browse masked keys, reveal_key when you need the raw secret, and add/update/delete for management. Custom-mode vaults require vault_key on reveal/add/update when changing values.

## Client Configuration Examples

### MCP Clients (Cursor, Claude Desktop, etc.)

```json
{
  "mcpServers": {
    "apivault": {
      "url": "https://mcp.apivault.tech/mcp"
    }
  }
}
```

OAuth completes automatically on first connection. No API keys in config.

## Connection Management

Users manage active MCP clients at:

**ApiVault → Settings → MCP Connections**

Each connection shows client name, granted scopes, and created/last-used timestamps. Revoking a connection invalidates its tokens immediately. MCP tokens are separate from CLI tokens issued via `apivault login`.

## Agent Best Practices

1. Start with `list_keys` — never assume key ids or environment names.
2. Call `reveal_key` only when the raw secret is strictly necessary for the task.
3. After revealing, write the secret to the user's env file or process config — not into the conversation.
4. For write operations, confirm the target environment and service with the user when ambiguous.
5. On auth failure, reconnect or complete OAuth once; do not retry tools in a loop without user interaction.
