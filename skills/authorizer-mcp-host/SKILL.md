---
name: authorizer-mcp-host
description: Connect an MCP host (Claude Code, Cursor, VS Code, any streamable-HTTP MCP client) to Authorizer's built-in remote MCP server at <url>/mcp, using either an audience-bound static bearer token or OAuth with client self-registration (CIMD / DCR). Use when the user says "connect Claude Code to Authorizer", "add Authorizer as an MCP server", "--mcp-enabled", "mcp_url", "MCP returns 401", "check_permissions tool", "dynamic client registration", or "client ID metadata document". Not for protecting your own MCP server (that is plain OAuth + JWT validation; see authorizer-backend-auth).
---

# Connect an MCP host to Authorizer's MCP server

Authorizer serves a read-only MCP tool surface over streamable HTTP at `POST <url>/mcp`:

| Tool | Auth | Purpose |
| --- | --- | --- |
| `meta` | no | Server feature flags |
| `profile` | yes | The caller's profile |
| `check_permissions` | yes | Batch `(relation, object)` checks (FGA) |
| `list_permissions` | yes | Objects/relations the caller can access |

The old stdio `authorizer mcp` subcommand is deprecated (removed in 2.5.0) — don't use it.

## 1. Enable it on the server

Add `--mcp-enabled` (and `--enable-client-id-metadata-document` for OAuth hosts) to the
server command from `authorizer-local-setup`. `--url` must be a real `http(s)` origin; the
MCP URL is `<url>/mcp`:

```bash
docker run -p 8080:8080 -v authorizer_data:/authorizer/data quay.io/authorizer/authorizer \
  --url=http://localhost:8080 --mcp-enabled --enable-client-id-metadata-document \
  ...   # the remaining flags from authorizer-local-setup
```

## 2. Understand the audience rule (the cause of most 401s)

`/mcp` accepts a token only when its `aud` is **exactly** `<url>/mcp`. A normal login
token (`aud` = client id) is rejected at `/mcp`, and an MCP token is rejected at
`/graphql`, `/v1/*` and gRPC. Get the right audience by passing
`resource=<url>/mcp` (RFC 8707) when requesting the token. Bearer header only — no
cookies, no admin secret.

## 3A. Static token (simplest, verified with Claude Code)

The token identifies a **service account**, not a human: permission checks resolve to
`service_account:<client_id>` and `profile` is not useful.

```bash
# 1. Create a service account (admin API; or dashboard → Clients). Secret is shown ONCE.
curl -s "$AUTHORIZER_URL/graphql" -H 'Content-Type: application/json' \
  -H "Origin: $AUTHORIZER_URL" -H "X-Authorizer-Admin-Secret: $ADMIN_SECRET" \
  -d '{"query":"mutation { _create_client(params: { name: \"mcp-host\", allowed_scopes: [\"openid\"] }) { client { client_id } client_secret } }"}'

# 2. Mint a token bound to the MCP resource
ACCESS_TOKEN=$(curl -s -X POST "$AUTHORIZER_URL/oauth/token" \
  -d grant_type=client_credentials \
  -d client_id=$SA_CLIENT_ID -d client_secret=$SA_CLIENT_SECRET \
  -d scope=openid \
  -d resource=$AUTHORIZER_URL/mcp | jq -r .access_token)

# 3. Register it in Claude Code
claude mcp add --transport http authorizer "$AUTHORIZER_URL/mcp" \
  --header "Authorization: Bearer $ACCESS_TOKEN"
```

The instance's own `client_id` is an interactive client and gets `unauthorized_client`
for `client_credentials` — use a service account. Machine tokens have no refresh token;
re-mint on expiry (`expires_in`, 1800 s by default) and update the header.

For an agent acting **for a user**, use an RFC 8693 delegated token with
`resource=<url>/mcp` instead (see `authorizer-agent-delegation`); its answers are
`perms(agent) ∩ perms(user)`, it lives 5 minutes and cannot be refreshed.

Other hosts (Cursor, VS Code): configure a streamable-HTTP server with URL
`<url>/mcp` and the same `Authorization: Bearer …` header in that host's MCP config.

## 3B. OAuth with self-registration (per-user identity)

The host discovers everything from the URL: `POST /mcp` without a token returns `401`
with `WWW-Authenticate: Bearer realm="authorizer", resource_metadata="<url>/.well-known/oauth-protected-resource/mcp"`;
the client then runs authorization code + PKCE with `resource=<url>/mcp`. The client
must be registered somehow — both mechanisms are **off by default** in plain flag mode:

| Flag | Use when |
| --- | --- |
| `--enable-client-id-metadata-document` | Preferred (MCP spec 2025-11-25). Client ID is an HTTPS URL to a JSON document |
| `--enable-dynamic-client-registration` | Client only supports RFC 7591 DCR. Older Claude Code (2.1.226) refused a server without `registration_endpoint`; 2.1.332 accepts a CIMD-only server (`claude mcp list` → *Needs authentication*). Add DCR if your host reports *does not support dynamic client registration* |
| `--client-id-metadata-allowed-domains` | Optional allow-list of hosts that may serve CIMD documents |

```bash
./authorizer --mcp-enabled --url=https://auth.example.com \
  --enable-client-id-metadata-document \
  --enable-dynamic-client-registration
```

Self-registered clients see a consent screen before a code is issued. DCR clients are
public only (`token_endpoint_auth_method: none`), S256 PKCE is required. Add
`--enable-dynamic-client-registration` if your host needs DCR.

Tell users to add the connector as `<url>/mcp` (with the path), not the bare origin.

## Verify

```bash
# Discovery challenge
curl -s -i -X POST "$AUTHORIZER_URL/mcp" | grep -i '^www-authenticate'   # Bearer realm="authorizer", resource_metadata=...
curl -s "$AUTHORIZER_URL/.well-known/oauth-protected-resource/mcp" | jq .resource   # "<url>/mcp"

# Tool list with the audience-bound token
curl -s -X POST "$AUTHORIZER_URL/mcp" \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | jq '[.result.tools[].name]'

# A normal login token must be rejected here (expect 401)
curl -s -o /dev/null -w '%{http_code}\n' -X POST "$AUTHORIZER_URL/mcp" \
  -H 'Content-Type: application/json' -H "Authorization: Bearer $USER_TOKEN" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

# DCR advertised (only if you enabled it)
curl -s "$AUTHORIZER_URL/.well-known/openid-configuration" | jq .registration_endpoint
```

In Claude Code, `claude mcp list` shows **Connected** for a static-token server, or
*Needs authentication* for an OAuth one until you sign in from `/mcp` in Claude Code.
Troubleshooting a 401: decode the token and check `aud` first — it must equal `<url>/mcp`.
Tool failures come back as results with `isError: true` (e.g. `Unauthenticated`,
`FailedPrecondition` when FGA is disabled).

Docs: https://docs.authorizer.dev/core/mcp
