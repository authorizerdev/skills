# AGENTS.md — Authorizer skills

Task guides for coding agents (Cursor, Codex, Gemini, Copilot, Claude Code) working with
[Authorizer](https://authorizer.dev), the open-source self-hosted auth server. Each guide
is a self-contained Markdown file with steps, verified snippets and a **Verify** section.
Read the one that matches the task before writing code.

| Task | Read |
| --- | --- |
| Run Authorizer locally, get `client_id` / secrets / URLs, wait for `/readyz`, create a test user | [`skills/authorizer-local-setup/SKILL.md`](skills/authorizer-local-setup/SKILL.md) |
| Add login/logout/protected routes to React or Next.js (`@authorizerdev/authorizer-react`) | [`skills/authorizer-react-integration/SKILL.md`](skills/authorizer-react-integration/SKILL.md) |
| Validate access tokens in a backend (Node, Go, Python SDKs, or JWKS) | [`skills/authorizer-backend-auth/SKILL.md`](skills/authorizer-backend-auth/SKILL.md) |
| Connect an MCP host (Claude Code, Cursor) to Authorizer's built-in `/mcp` server | [`skills/authorizer-mcp-host/SKILL.md`](skills/authorizer-mcp-host/SKILL.md) |
| Model permissions with OpenFGA: model, tuples, `check_permissions`, agents | [`skills/authorizer-fga/SKILL.md`](skills/authorizer-fga/SKILL.md) |
| Let an agent act for a user (RFC 8693 token exchange, `act` claim) | [`skills/authorizer-agent-delegation/SKILL.md`](skills/authorizer-agent-delegation/SKILL.md) |

Ground rules that apply to every task:

- Authorizer v2 is configured **only by CLI flags** — no `.env`, no environment variables.
- State-changing HTTP calls (`POST /graphql`, `POST /v1/*`) need `Content-Type: application/json`
  and an allowed `Origin` header, or they fail with `403 csrf_validation_failed`.
- Admin operations (prefixed `_`, e.g. `_fga_write_tuples`, `_create_client`) use the
  `X-Authorizer-Admin-Secret` header and must run server-side only.
- Never commit `client_secret`, the admin secret, or the `.authorizer/` dev state directory.
- Do not invent flags, endpoints or SDK functions. When something isn't covered here,
  check https://docs.authorizer.dev/llms.txt (index) or https://docs.authorizer.dev/llms-full.txt (full text).
