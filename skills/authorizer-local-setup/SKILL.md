---
name: authorizer-local-setup
description: Run a local Authorizer v2 auth server for development and read back its connection details (URL, client ID, client secret, admin secret, GraphQL/REST/JWKS/MCP URLs). Use when the user says "run Authorizer locally", "start an auth server", "set up Authorizer for development", "I need a client_id / admin secret", "authorizer dev", "--print-config", or before any other Authorizer skill needs a running instance. Covers `authorizer dev` (zero-flag mode with generated secrets) with a fallback to explicit CLI flags, waiting for /readyz, and creating a test user.
---

# Run Authorizer locally

Authorizer v2 is configured **only by CLI flags** — it does not read `.env` files or
environment variables. Pick path A if the binary has the `dev` subcommand, else path B.

## 1. Detect which path you have

```bash
./authorizer dev --help    # exit 0 and "Run a local development server..." → path A
                           # 'unknown command "dev"' → path B
```

`authorizer dev` arrives in the first release after 2.4.9. On older release binaries
and images use path B. To get path A before then, build from source (Go toolchain
required):

```bash
git clone https://github.com/authorizerdev/authorizer.git && cd authorizer
go build -o build/authorizer .
# Optional, only if you need the built-in login UI / dashboard: make build-app build-dashboard
```

## 2A. `authorizer dev` (preferred)

Run it **from the directory that contains `web/`** (the repo root or an extracted release
archive) — the login UI, dashboard and email templates load from there.

```bash
./build/authorizer dev --print-config   # prints connection JSON and exits; does not start the server
./build/authorizer dev                   # starts the server
```

On first run it generates a client ID/secret, admin secret, encryption key and an RS256
key pair into `.authorizer/` (mode 0600, git-ignored) and reuses them on every later run.
Defaults (each only when you don't pass the flag): SQLite at `.authorizer/authorizer.db`,
`--host=127.0.0.1`, `--url=http://localhost:<http-port>`,
`--allowed-origins=<url>,http://localhost:3000,http://localhost:5173,http://localhost:4200`,
`--mcp-enabled=true`, `--enable-client-id-metadata-document=true`. Every other server flag
still works and wins: if your frontend runs on another port, pass
`--allowed-origins=<url>,<your app origin>` (list the server's own URL too).

`--print-config` computes values from the flags you give it, so pass the same flags you
start the server with (e.g. `--http-port`). The same JSON is written to
`.authorizer/dev.json` on every start. Fields: `url`, `login_url`, `dashboard_url`,
`graphql_url`, `rest_base_url`, `openapi_url`, `grpc_address`,
`openid_configuration_url`, `jwks_url`, `mcp_url`, `allowed_origins`, `client_id`, `client_secret`,
`admin_secret`, `admin_secret_header`, `database_type`, `database_url`, `state_dir`.

Load them into the shell for the other skills:

```bash
CFG=.authorizer/dev.json   # or: ./build/authorizer dev --print-config > /tmp/authz.json
export AUTHORIZER_URL=$(jq -r .url "$CFG")
export CLIENT_ID=$(jq -r .client_id "$CFG")
export CLIENT_SECRET=$(jq -r .client_secret "$CFG")
export ADMIN_SECRET=$(jq -r .admin_secret "$CFG")
```

Never commit `.authorizer/` or paste `client_secret` / `admin_secret` into source code.
`--dev-dir` moves the state; it must be new, empty, or a previous `dev` directory.

## 2B. Explicit flags (any release)

From the docs' "Minimal configuration for local development":

```bash
./authorizer \
  --env=development \
  --http-port=8080 \
  --database-type=sqlite \
  --database-url=test.db \
  --url=http://localhost:8080 \
  --client-id=123456 \
  --client-secret=secret \
  --admin-secret=admin \
  --jwt-type=HS256 \
  --jwt-secret=test \
  --encryption-key=test-encryption-key \
  --allowed-origins=http://localhost:3000,http://localhost:8080
```

Then `AUTHORIZER_URL=http://localhost:8080 CLIENT_ID=123456 CLIENT_SECRET=secret ADMIN_SECRET=admin`.

Docker equivalent: `docker run -p 8080:8080 quay.io/authorizer/authorizer:latest <same flags>`.
Omit `--encryption-key` on releases before 2.4.0 (they don't have it).

Notes that bite:
- `--allowed-origins` is an exact allow-list for CSRF. Include **your app's origin and
  Authorizer's own URL**; with only `http://localhost:3000`, requests carrying
  `Origin: http://localhost:8080` get `403 csrf_validation_failed`. The default (`*`)
  accepts only `Origin` equal to the server's own host, so a browser app on another
  port needs an explicit list.
- HS256 publishes **no keys** at `/.well-known/jwks.json` (`{"keys":[]}`). If a backend
  must verify tokens offline via JWKS, use path A (RS256) or configure an RS*/ES* key.
- `--enable-email-verification` defaults to `false`; turning it on without SMTP flags is a
  fatal boot error (2.4.0+).

## 3. Wait for readiness

```bash
until [ "$(curl -s -o /dev/null -w '%{http_code}' "$AUTHORIZER_URL/readyz")" = 200 ]; do sleep 0.5; done
```

`/readyz` returns `{"status":"ready"}` (200) or 503. `/healthz` is the liveness probe.

## 4. Create a test user and get an access token

MFA setup is **offered by default since 2.4.0**, so signup/login return no access token
(`"Proceed to mfa setup"`) until the user enrols or skips. For scripted local testing,
start the server with `--disable-mfa` (works with both paths). State-changing requests
need `Content-Type: application/json` **and** an allowed `Origin` header.

```bash
curl -s -X POST "$AUTHORIZER_URL/v1/signup" \
  -H 'Content-Type: application/json' -H "Origin: $AUTHORIZER_URL" \
  -d '{"email":"alice@example.com","password":"Alice-pass-123!","confirm_password":"Alice-pass-123!"}'

export USER_TOKEN=$(curl -s -X POST "$AUTHORIZER_URL/v1/login" \
  -H 'Content-Type: application/json' -H "Origin: $AUTHORIZER_URL" \
  -d '{"email":"alice@example.com","password":"Alice-pass-123!"}' | jq -r .access_token)
```

Built-in UIs: `$AUTHORIZER_URL/app` (login) and `$AUTHORIZER_URL/dashboard` (admin; log in
with the admin secret).

## Verify

```bash
curl -s "$AUTHORIZER_URL/readyz"                                   # {"status":"ready"}
curl -s "$AUTHORIZER_URL/.well-known/openid-configuration" | jq .issuer   # equals $AUTHORIZER_URL
curl -s "$AUTHORIZER_URL/.well-known/jwks.json" | jq '.keys | length'     # 1 on path A (RS256), 0 on HS256
curl -s -X POST "$AUTHORIZER_URL/v1/validate_jwt_token" \
  -H 'Content-Type: application/json' -H "Origin: $AUTHORIZER_URL" \
  -d "{\"token_type\":\"access_token\",\"token\":\"$USER_TOKEN\"}" | jq .is_valid   # true
```

`authorizer dev` is for local development only — its secrets sit in a plain file. For
shared or production deployments pass real secrets via flags
(https://docs.authorizer.dev/core/server-config).
