---
name: authorizer-fga
description: Model fine-grained permissions with Authorizer's embedded OpenFGA (ReBAC) engine — write an authorization model in the OpenFGA DSL, grant access with relationship tuples, and check access with check_permissions / list_permissions over GraphQL, REST or the JS/Go/Python SDKs, including AI agents (type agent, perms(agent) ∩ perms(user)) and service accounts. Use when the user says "add permissions", "who can view this document", "RBAC/ReBAC", "OpenFGA", "FGA model", "write tuples", "share a document", "check_permissions", "list what a user can access", "permission-aware RAG", or "restrict what an agent can see".
---

# Fine-grained authorization (OpenFGA) in Authorizer

Two touchpoints, two credentials:

| Step | API | Credential |
| --- | --- | --- |
| Author model, write/delete tuples | `_fga_write_model`, `_fga_write_tuples`, `_fga_delete_tuples` | Admin secret — **server-side only** |
| Check access | `check_permissions`, `list_permissions` | The **caller's own token** |

FGA is on by default with a SQL main database (SQLite, Postgres, MySQL, …). With a NoSQL
main database pass `--fga-store sqlite|postgres|mysql|memory` and `--fga-store-url`.
When disabled, FGA calls return `fine-grained authorization is not enabled`.

All admin `POST /graphql` calls need `Content-Type: application/json`, an allowed
`Origin` header, and `X-Authorizer-Admin-Secret`.

## 1. Write the model

```dsl
model
  schema 1.1

type user
type agent

type document
  relations
    define owner: [user]
    define editor: [user] or owner
    define viewer: [user, agent] or editor
    define can_view: viewer
    define can_edit: editor
    define can_delete: owner
```

Rules: subjects are `user:<id>` with the Authorizer user id (token `sub`), never email.
Add `type service_account` (and admit it, e.g. `[user, service_account]`) for
`client_credentials` callers, which resolve to `service_account:<client_id>`. Add
`type agent` for delegated agents (step 4). Supported: `or`, `and`, `but not`,
`X from parent`, `team#member` usersets, `user:*`. **Not supported**: OpenFGA conditions
(`with …`) — use contextual tuples instead. Saving creates a new immutable active version.

```bash
curl -s "$AUTHORIZER_URL/graphql" -H 'Content-Type: application/json' \
  -H "Origin: $AUTHORIZER_URL" -H "X-Authorizer-Admin-Secret: $ADMIN_SECRET" \
  -d '{
    "query": "mutation ($params: FgaWriteModelInput!) { _fga_write_model(params: $params) { id } }",
    "variables": { "params": { "dsl": "model\n  schema 1.1\n\ntype user\ntype agent\n\ntype document\n  relations\n    define owner: [user]\n    define editor: [user] or owner\n    define viewer: [user, agent] or editor\n    define can_view: viewer\n    define can_edit: editor\n    define can_delete: owner" } }
  }'
```

(The dashboard — `/dashboard` → Authorization — does the same with a roles matrix or DSL editor.)

## 2. Write tuples on your domain events (server-side)

```js
async function writeTuples(tuples) {
  await fetch(`${process.env.AUTHORIZER_URL}/graphql`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Origin: process.env.AUTHORIZER_URL,
      'X-Authorizer-Admin-Secret': process.env.AUTHORIZER_ADMIN_SECRET,
    },
    body: JSON.stringify({
      query: `mutation ($params: FgaWriteTuplesInput!) { _fga_write_tuples(params: $params) { message } }`,
      variables: { params: { tuples } },
    }),
  });
}

// On "document created":
await writeTuples([{ user: `user:${creatorId}`, relation: 'owner', object: `document:${docId}` }]);
```

Revoke with `_fga_delete_tuples` and the same tuple shape. Tuples are validated against
the active model.

## 3. Check access (caller's token)

GraphQL / REST (`POST /v1/check_permissions`, 1–100 checks, results in order):

```bash
curl -s -X POST "$AUTHORIZER_URL/v1/check_permissions" \
  -H "Authorization: Bearer $USER_TOKEN" -H 'Content-Type: application/json' \
  -H "Origin: $AUTHORIZER_URL" \
  -d '{"checks":[{"relation":"can_view","object":"document:1"},{"relation":"can_edit","object":"document:1"}]}'
```

JS (`@authorizerdev/authorizer-js`; in Node also set `extraHeaders: { Origin: authorizerURL }`):

```js
const { data } = await auth.checkPermissions(
  { checks: [{ relation: 'can_view', object: `document:${id}` }] },
  { Authorization: req.headers.authorization },
);
if (!data?.results?.[0]?.allowed) return res.status(403).json({ error: 'forbidden' });

const { data: list } = await auth.listPermissions(
  { relation: 'can_view', object_type: 'document' },
  { Authorization: req.headers.authorization },
); // list.objects => ['document:1', ...]; list.truncated if > 1000
```

Go:

```go
res, err := client.CheckPermissions(&authorizer.CheckPermissionsRequest{
	Checks: []*authorizer.PermissionCheckInput{{Relation: "can_view", Object: "document:1"}},
}, map[string]string{"Authorization": "Bearer " + token})
if err != nil || len(res.Results) == 0 || !res.Results[0].Allowed { /* deny */ }
```

Python:

```python
from authorizer import CheckPermissionsRequest, PermissionCheckInput
res = client.check_permissions(
    CheckPermissionsRequest(checks=[PermissionCheckInput(relation="can_view", object="document:1")]),
    headers={"Authorization": f"Bearer {token}"},
)
allowed = res.results[0].allowed
```

The subject is pinned from the token. An explicit `user` is honored only for super-admins
or when it equals the caller's own subject. Treat any error as deny. For list pages, call
`list_permissions` once and filter your DB query — don't loop `check_permissions`.

## 4. AI agents: perms(agent) ∩ perms(user)

Declaring `type agent` in the model **is** the opt-in. A delegated token (RFC 8693, see
`authorizer-agent-delegation`) is then checked as both `agent:<agent client_id>` and
`user:<sub>`; both must allow. Grant the agent separately:

```graphql
mutation {
  _fga_write_tuples(params: { tuples: [
    { user: "user:ALICE_ID", relation: "viewer", object: "document:q4-plan" }
    { user: "agent:AGENT_CLIENT_ID", relation: "viewer", object: "document:q4-plan" }
  ]}) { message }
}
```

Grant agents **before** deploying a model with `type agent` — from that moment every
delegated check needs the agent half. Without `type agent`, delegated checks are denied
(2.4.0+). To cut an agent off immediately, delete its tuples (deactivating its service
account doesn't stop already-issued tokens). Metric
`authorizer_fga_delegated_checks_total{outcome}` distinguishes `denied_by_agent`,
`denied_by_user`, `not_enforced`.

## Verify

```bash
# Model installed
curl -s "$AUTHORIZER_URL/graphql" -H 'Content-Type: application/json' -H "Origin: $AUTHORIZER_URL" \
  -H "X-Authorizer-Admin-Secret: $ADMIN_SECRET" -d '{"query":"{ _fga_get_model { id } }"}' | jq .data

# Owner/viewer allowed, unrelated object denied
curl -s -X POST "$AUTHORIZER_URL/v1/check_permissions" -H "Authorization: Bearer $USER_TOKEN" \
  -H 'Content-Type: application/json' -H "Origin: $AUTHORIZER_URL" \
  -d '{"checks":[{"relation":"can_view","object":"document:q4-plan"},{"relation":"can_view","object":"document:nope"}]}' \
  | jq '[.results[].allowed]'   # [true,false]
```

With a delegated token, a document granted to the user but not the agent must return
`allowed: false`. Debug a decision with admin-only `_fga_expand` / `_fga_list_users`.

Docs: https://docs.authorizer.dev/core/authorization · https://docs.authorizer.dev/core/fga-guide ·
https://docs.authorizer.dev/enterprise/agent-identity
