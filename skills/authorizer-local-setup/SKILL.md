---
name: authorizer-local-setup
description: Run a local Authorizer v2 auth server for development with Docker (or the binary) and know its connection details (URL, client ID, client secret, admin secret, GraphQL/REST/JWKS/MCP URLs). Use when the user says "run Authorizer locally", "start an auth server", "set up Authorizer for development", "docker run authorizer", "I need a client_id / admin secret", or before any other Authorizer skill needs a running instance. Covers the Docker quick start, RS256 keys for JWKS validation, allowed origins for a frontend, waiting for /readyz, and creating a test user.
---

# Run Authorizer locally

Authorizer v2 is configured **only by CLI flags** — it does not read `.env` files or
environment variables. Docker is the recommended way to run it locally: the image ships
the login UI and dashboard, and needs no Go toolchain.

## 1. Start it with Docker

```bash
docker run -p 8080:8080 \
  -v authorizer_data:/authorizer/data \
  quay.io/authorizer/authorizer \
  --database-type=sqlite \
  --database-url=/authorizer/data/data.db \
  --url=http://localhost:8080 \
  --client-id=123456 \
  --client-secret=secret \
  --admin-secret=admin \
  --jwt-type=HS256 \
  --jwt-secret=test \
  --encryption-key=test-encryption-key
```

```bash
export AUTHORIZER_URL=http://localhost:8080 CLIENT_ID=123456 CLIENT_SECRET=secret ADMIN_SECRET=admin
```

These values are placeholders for local use only — never reuse them anywhere shared.
The `authorizer_data` named volume keeps the SQLite database across restarts. The image
runs as a non-root user that owns `/authorizer/data` (v2.4.9+), so no `-u root` is
needed; on older images add `-u root`.

Add flags to the same command as your task needs:

| Need | Add |
| --- | --- |
| A frontend on another origin (e.g. React on `:3000`) | `--allowed-origins=http://localhost:8080,http://localhost:3000` |
| Tokens straight from signup/login in scripts | `--disable-mfa` |
| Authorizer's MCP server (`authorizer-mcp-host`) | `--mcp-enabled --enable-client-id-metadata-document` |
| JWKS validation by a backend or MCP server | RS256 keys, see section 2 |
| Another host port, e.g. 9000 | `-p 9000:8080` and `--url=http://localhost:9000` |

Notes that bite:
- `--url` is required; the server refuses to start without it. It must be the URL
  clients use, so with `-p 9000:8080` it is `http://localhost:9000`.
- `--allowed-origins` is an exact allow-list for CSRF and redirects. Include **your
  app's origin and Authorizer's own URL**; with only `http://localhost:3000`, the
  built-in login page gets `403`. The default (`*`) accepts state-changing requests only
  from the server's own origin, so a browser app on another port needs an explicit list.
- Inside the container `localhost` is the container. To reach a database on your
  machine use `host.docker.internal` (Docker Desktop), or on Linux add
  `--add-host=host.docker.internal:host-gateway`.
- `--enable-email-verification` defaults to `false`; turning it on without SMTP flags is
  a fatal boot error (2.4.0+).

## 2. RS256 keys (needed for JWKS)

HS256 publishes **no keys** at `/.well-known/jwks.json` (`{"keys":[]}`). Anything that
verifies tokens offline through JWKS needs an asymmetric key:

```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

docker run -p 8080:8080 \
  -v authorizer_data:/authorizer/data \
  quay.io/authorizer/authorizer \
  --database-type=sqlite \
  --database-url=/authorizer/data/data.db \
  --url=http://localhost:8080 \
  --client-id=123456 \
  --client-secret=secret \
  --admin-secret=admin \
  --jwt-type=RS256 \
  --jwt-private-key="$(cat private.pem)" \
  --jwt-public-key="$(cat public.pem)" \
  --encryption-key=test-encryption-key
```

`--encryption-key` is required with `RS*`/`ES*` (there is no `--jwt-secret` to fall back
to). Keep `private.pem` out of version control.

## 3. Without Docker

Recent releases publish the Docker image, not binaries, so build from source (Go and
Node required) and run **from the repository root** — the login UI, dashboard and email
templates load from its `web/` folder:

```bash
git clone https://github.com/authorizerdev/authorizer.git && cd authorizer
go build -o build/authorizer . && make build-app build-dashboard
./build/authorizer --database-type=sqlite --database-url=authorizer.db \
  --url=http://localhost:8080 --client-id=123456 --client-secret=secret \
  --admin-secret=admin --jwt-type=HS256 --jwt-secret=test \
  --encryption-key=test-encryption-key
```

## 4. Wait for readiness

```bash
until [ "$(curl -s -o /dev/null -w '%{http_code}' "$AUTHORIZER_URL/readyz")" = 200 ]; do sleep 0.5; done
```

`/readyz` returns `{"status":"ready"}` (200) or 503. `/healthz` is the liveness probe.

## 5. Create a test user and get an access token

MFA setup is **offered by default since 2.4.0**, so signup/login return no access token
(`"Proceed to mfa setup"`) until the user enrols or skips. For scripted local testing,
start the server with `--disable-mfa`. State-changing requests need
`Content-Type: application/json` **and** an allowed `Origin` header.

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
curl -s "$AUTHORIZER_URL/.well-known/jwks.json" | jq '.keys | length'     # 1 with RS256, 0 with HS256
curl -s -X POST "$AUTHORIZER_URL/v1/validate_jwt_token" \
  -H 'Content-Type: application/json' -H "Origin: $AUTHORIZER_URL" \
  -d "{\"token_type\":\"access_token\",\"token\":\"$USER_TOKEN\"}" | jq .is_valid   # true
```

For shared or production deployments pass real secrets
(https://docs.authorizer.dev/core/server-config, https://docs.authorizer.dev/deployment/docker).
