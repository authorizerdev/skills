# Authorizer agent skills

Agent skills for [Authorizer](https://authorizer.dev) — the open-source, self-hosted
authentication and authorization server (OAuth2/OIDC, social login, MFA, passkeys,
OpenFGA fine-grained authorization, a built-in MCP server, and RFC 8693 agent delegation).

Each skill teaches a coding agent one task end to end, with snippets checked against the
Authorizer docs and source and a **Verify** section of concrete checks:

| Skill | What it does |
| --- | --- |
| [`authorizer-local-setup`](skills/authorizer-local-setup/SKILL.md) | Run a local instance (`authorizer dev` or explicit flags), read connection details, wait for `/readyz`, create a test user |
| [`authorizer-react-integration`](skills/authorizer-react-integration/SKILL.md) | Add login to React / Next.js with `@authorizerdev/authorizer-react` |
| [`authorizer-backend-auth`](skills/authorizer-backend-auth/SKILL.md) | Validate access tokens in Node, Go or Python, or offline via JWKS |
| [`authorizer-mcp-host`](skills/authorizer-mcp-host/SKILL.md) | Connect Claude Code / Cursor to Authorizer's `/mcp` server (static token or OAuth with CIMD/DCR) |
| [`authorizer-fga`](skills/authorizer-fga/SKILL.md) | Model permissions with OpenFGA, write tuples, check access — including AI agents |
| [`authorizer-agent-delegation`](skills/authorizer-agent-delegation/SKILL.md) | RFC 8693 token exchange so an agent can act for a user |

## Install

### Claude Code

```text
/plugin marketplace add authorizerdev/skills
/plugin install authorizer@authorizer-skills
```

Skills load automatically when a task matches their description (for example "add
Authorizer login to my Next.js app").

### Cursor, Codex and other agents

These tools read `AGENTS.md`. Either:

- copy [`AGENTS.md`](AGENTS.md) and the `skills/` directory into your project root, or
- reference it from your own `AGENTS.md` / Cursor rules, e.g.
  "For Authorizer tasks, follow https://github.com/authorizerdev/skills/blob/main/AGENTS.md".

Each `SKILL.md` is plain Markdown with YAML front matter, so any agent can read it directly.

## Docs for LLMs

The Authorizer docs publish [llms.txt](https://llmstxt.org) files:

- Index: https://docs.authorizer.dev/llms.txt
- Full text: https://docs.authorizer.dev/llms-full.txt

To give an MCP host live access to the docs, run the
[`mcpdoc`](https://github.com/langchain-ai/mcpdoc) llms.txt server over stdio:

```bash
uvx --with 'mcp<2' --from mcpdoc mcpdoc --urls Authorizer:https://docs.authorizer.dev/llms.txt
```

For example, in Claude Code:

```bash
claude mcp add authorizer-docs -- uvx --with 'mcp<2' --from mcpdoc mcpdoc --urls Authorizer:https://docs.authorizer.dev/llms.txt
```

`--with 'mcp<2'` pins the MCP Python SDK: the current `mcpdoc` release fails at startup
against `mcp` 2.x (`No module named 'mcp.server.fastmcp'`). Once `mcpdoc` supports 2.x,
`uvx --from mcpdoc mcpdoc --urls ...` is enough.

## Note on `authorizer dev`

`authorizer dev` (zero-flag local server with `--print-config`) arrives in the first
release after 2.4.9. The local-setup skill detects it with `authorizer dev --help` and
falls back to the documented explicit flags otherwise.

## Contributing

Every command, flag, endpoint and SDK call in these skills must match the
[Authorizer docs](https://docs.authorizer.dev) or source. If you can't confirm it, leave
it out. Keep each `SKILL.md` under about 250 lines and keep its **Verify** section runnable.

## License

[Apache-2.0](LICENSE), same as Authorizer.
