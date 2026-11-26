---
name: authorizer-agent-delegation
description: Let an AI agent or service act on behalf of a user with Authorizer's RFC 8693 token exchange — register the agent as a service_account client, get its own client_credentials token, exchange the user's token plus the agent's token for a short-lived, down-scoped, audience-bound delegated token with an act claim, and validate it at a resource server. Use when the user says "agent acting on behalf of a user", "token exchange", "RFC 8693", "on-behalf-of", "delegated token", "act claim", "actor_token", "subject_token", "agent needs the user's permissions", or "multi-hop agent delegation".
---

# Agent delegation with RFC 8693 token exchange

Authorizer implements a **delegation-only** profile: the result carries both identities —
`sub` = the user (delegated subject), `act.sub` = the agent's `client_id` — and is:
5-minute TTL, no refresh token, exactly one audience (`resource`), scope
`= subject.scope ∩ agent.allowed_scopes (∩ requested scope)`.

Prerequisites: a running Authorizer (`authorizer-local-setup`), `ADMIN_SECRET`, and a
user access token `USER_TOKEN`.

## 1. Register the agent as a service account

The instance's own `client_id` is an interactive client and cannot do this. Create a
`service_account`; its `allowed_scopes` is the agent's **delegation ceiling** (empty =
can delegate nothing).

```bash
curl -s "$AUTHORIZER_URL/graphql" -H 'Content-Type: application/json' \
  -H "Origin: $AUTHORIZER_URL" -H "X-Authorizer-Admin-Secret: $ADMIN_SECRET" \
  -d '{"query":"mutation { _create_client(params: { name: \"calendar-agent\", allowed_scopes: [\"openid\", \"profile\", \"email\"] }) { client { id client_id allowed_scopes } client_secret } }"}'
```

`client_secret` is returned **once** — store it in a secret manager. Export
`AGENT_CLIENT_ID` and `AGENT_CLIENT_SECRET`. Rotate with `_rotate_client_secret`, disable
with `_update_client(params: { id, is_active: false })`.

## 2. Agent gets its own token

```bash
AGENT_TOKEN=$(curl -s -X POST "$AUTHORIZER_URL/oauth/token" \
  -u "$AGENT_CLIENT_ID:$AGENT_CLIENT_SECRET" \
  -d "grant_type=client_credentials" | jq -r .access_token)
```

## 3. Exchange user token + agent token → delegated token

```bash
curl -s -X POST "$AUTHORIZER_URL/oauth/token" \
  -u "$AGENT_CLIENT_ID:$AGENT_CLIENT_SECRET" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
  -d "subject_token=$USER_TOKEN" \
  -d "subject_token_type=urn:ietf:params:oauth:token-type:access_token" \
  -d "actor_token=$AGENT_TOKEN" \
  -d "actor_token_type=urn:ietf:params:oauth:token-type:access_token" \
  -d "resource=https://calendar.example" \
  -d "scope=openid"
```

Response: `access_token`, `issued_token_type`
(`urn:ietf:params:oauth:token-type:access_token`), `token_type: Bearer`,
`expires_in: 300`, `scope`.

Parameter rules:
- `actor_token` is **mandatory** (subject-only exchange = impersonation → `invalid_request`),
  and its `sub` must be the authenticated agent.
- `resource` is required, exactly once. Pick the audience by where the token will be used:

| Token will be presented at | `resource` |
| --- | --- |
| Your own API / resource server | that API's URI, e.g. `https://calendar.example` |
| Authorizer's GraphQL / REST / gRPC (e.g. `check_permissions`) | `$AUTHORIZER_URL` |
| Authorizer's MCP server | `$AUTHORIZER_URL/mcp` |

These audiences are not interchangeable: a token bound to one is refused at the others.
- `scope` is optional; the result is never wider than the user's token or the agent's
  ceiling. Empty intersection → `invalid_scope`.
- Client auth: `client_secret_basic` (`-u`), `client_secret_post`, or an RFC 7523
  `client_assertion` — never more than one.
- Multi-hop: a delegated token can be the next `subject_token`; `act` nests; max depth 4.

## 4. Validate at your resource server (offline)

Verify with JWKS (RS*/ES* keys required — see `authorizer-local-setup`, section 2), checking `iss`,
`aud` = your resource URI, `exp`, and `scope`:

```js
import { createRemoteJWKSet, jwtVerify } from 'jose';

const oidc = await (await fetch(`${AUTHORIZER_URL}/.well-known/openid-configuration`)).json();
const jwks = createRemoteJWKSet(new URL(oidc.jwks_uri));
const { payload } = await jwtVerify(token, jwks, {
  issuer: oidc.issuer,
  audience: 'https://calendar.example',
});
// payload.sub = user, payload.act.sub = agent, payload.scope = granted scopes
```

Do **not** use `/oauth/introspect` for delegated tokens — it always returns
`{"active": false}` to a resource server (the token's `aud` is not the caller's
`client_id`). Offline verification cannot see logout/revocation; the 5-minute TTL is the
bound, so re-exchange instead of caching long-lived.

## 5. Agent permissions at Authorizer

Exchanged with `resource=$AUTHORIZER_URL`, the token can call `check_permissions` /
`list_permissions`, `profile`, `meta` (scope `openid`). Mutations need extra scopes
(`authorizer:profile:write`, `authorizer:account:delete`) present on **both** the user
token and the agent ceiling; everything else is denied to delegated callers. With
`type agent` in the FGA model, answers are `perms(agent) ∩ perms(user)` — see
`authorizer-fga` step 4.

## Errors

| Error | Usual cause |
| --- | --- |
| `invalid_request` | Missing `actor_token`, bad token type URN, `resource` count ≠ 1, chain > 4 |
| `invalid_grant` | Subject/actor token expired or invalid, actor token not the caller's, user revoked |
| `invalid_scope` | Agent has no ceiling, or the scope intersection is empty |
| `invalid_client` / `unauthorized_client` | Bad credentials, or caller is not a `service_account` |

## Verify

```bash
DELEGATED=$(curl -s -X POST "$AUTHORIZER_URL/oauth/token" -u "$AGENT_CLIENT_ID:$AGENT_CLIENT_SECRET" \
  -d grant_type=urn:ietf:params:oauth:grant-type:token-exchange \
  -d subject_token=$USER_TOKEN -d subject_token_type=urn:ietf:params:oauth:token-type:access_token \
  -d actor_token=$AGENT_TOKEN -d actor_token_type=urn:ietf:params:oauth:token-type:access_token \
  -d resource=$AUTHORIZER_URL | jq -r .access_token)

# Claims: sub = user id, act.sub = agent client_id, aud = resource
echo "$DELEGATED" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null; echo

# Works at Authorizer (resource=$AUTHORIZER_URL)
curl -s -X POST "$AUTHORIZER_URL/v1/check_permissions" -H "Authorization: Bearer $DELEGATED" \
  -H 'Content-Type: application/json' -H "Origin: $AUTHORIZER_URL" \
  -d '{"checks":[{"relation":"can_view","object":"document:q4-plan"}]}'

# Impersonation is refused (expect "invalid_request")
curl -s -X POST "$AUTHORIZER_URL/oauth/token" -u "$AGENT_CLIENT_ID:$AGENT_CLIENT_SECRET" \
  -d grant_type=urn:ietf:params:oauth:grant-type:token-exchange \
  -d subject_token=$USER_TOKEN -d subject_token_type=urn:ietf:params:oauth:token-type:access_token \
  -d resource=$AUTHORIZER_URL | jq .error
```

Runnable examples: https://github.com/authorizerdev/examples/tree/main/with-agent-delegation ·
https://github.com/authorizerdev/examples/tree/main/with-agent-permissions

Docs: https://docs.authorizer.dev/enterprise/token-exchange · https://docs.authorizer.dev/enterprise/agent-identity ·
https://docs.authorizer.dev/core/client-registry
