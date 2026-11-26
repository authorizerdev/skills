---
name: authorizer-backend-auth
description: Validate Authorizer-issued access tokens in a backend API (Node/Express with @authorizerdev/authorizer-js, Go with authorizer-go, Python with authorizer-py) or offline against the JWKS endpoint (jose). Use when the user says "protect my API with Authorizer", "validate the bearer token", "verify JWT from Authorizer", "auth middleware", "check the Authorization header", "JWKS validation", or "validate_jwt_token". Also covers issuer/audience checks and the CSRF Origin header Node callers need.
---

# Validate Authorizer tokens in a backend

Prerequisite: a running Authorizer (`authorizer-local-setup`), its URL and `client_id`.

## Choose a method

| Method | Network call per request | Sees revocation/logout | Needs |
| --- | --- | --- | --- |
| SDK `validate_jwt_token` | yes | yes (server-side check) | `client_id`, server URL |
| Offline JWKS verify | no (keys cached) | no — only `exp` bounds it | RS*/ES* signing (HS* publishes no keys) |

Access-token claims you will see: `iss` (= `--url`), `sub` (user id), `aud`, `client_id`,
`scope`, `roles`, `allowed_roles`, `exp`, `token_type: "access_token"`. `aud` is the
instance `client_id` for normal logins, or the RFC 8707 `resource` when one was requested
— read `client_id` to learn which client asked.

## Node / Express (authorizer-js)

```bash
npm install @authorizerdev/authorizer-js
```

```js
import { Authorizer } from '@authorizerdev/authorizer-js';

const authorizerURL = process.env.AUTHORIZER_URL;
const auth = new Authorizer({
  authorizerURL,
  redirectURL: authorizerURL,
  clientID: process.env.CLIENT_ID,
  // POST /graphql is CSRF-protected and Node sends no Origin by itself.
  extraHeaders: { Origin: authorizerURL },
});

export async function requireAuth(req, res, next) {
  const [scheme, token] = (req.headers.authorization || '').split(' ');
  if (scheme?.toLowerCase() !== 'bearer' || !token) return res.status(401).json({ error: 'unauthorized' });
  const { data, errors } = await auth.validateJWTToken({ token_type: 'access_token', token });
  if (errors.length || !data?.is_valid) return res.status(401).json({ error: 'unauthorized' });
  req.user = data.claims;
  next();
}
```

Without `extraHeaders.Origin` every call fails with `HTTP 403`. `Origin` must be
Authorizer's own URL (default `--allowed-origins=*`) or an entry in `--allowed-origins`.

## Go (authorizer-go)

```bash
go get github.com/authorizerdev/authorizer-go/v2
```

```go
import authorizer "github.com/authorizerdev/authorizer-go/v2"

client, err := authorizer.NewAuthorizerClient(clientID, authorizerURL, "", nil)
if err != nil { /* handle */ }

func requireAuth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		token := strings.TrimPrefix(r.Header.Get("Authorization"), "Bearer ")
		res, err := client.ValidateJWTToken(&authorizer.ValidateJWTTokenRequest{
			TokenType: authorizer.TokenTypeAccessToken,
			Token:     token,
		})
		if err != nil || !res.IsValid {
			http.Error(w, "unauthorized", http.StatusUnauthorized) // fail closed
			return
		}
		next.ServeHTTP(w, r) // res.Claims holds the decoded claims
	})
}
```

`NewAuthorizerClient(clientID, authorizerURL, redirectURL, extraHeaders)`. Token types:
`TokenTypeAccessToken`, `TokenTypeIDToken`, `TokenTypeRefreshToken`.

## Python (authorizer-py)

```bash
pip install --pre authorizer-py   # 0.3.0 pre-release is needed for server 2.4.0+
```

```python
from authorizer import AuthorizerClient, ValidateJWTTokenRequest, TokenType

client = AuthorizerClient(client_id=CLIENT_ID, authorizer_url=AUTHORIZER_URL)  # no trailing slash

def is_valid(bearer: str) -> bool:
    res = client.validate_jwt_token(ValidateJWTTokenRequest(
        token=bearer, token_type=TokenType.ACCESS_TOKEN,
    ))
    return res.is_valid   # res.claims has the decoded claims
```

The Python and Go SDKs set the CSRF `Origin` header themselves. An
`AsyncAuthorizerClient` with the same methods (awaited) exists for async frameworks.

## Offline JWKS verification (Node, `jose`)

Requires an asymmetric key (RS256 setup in `authorizer-local-setup`, section 2); HS256
publishes an empty JWKS and this cannot work.

```js
import { createRemoteJWKSet, jwtVerify } from 'jose';

const oidc = await (await fetch(`${AUTHORIZER_URL}/.well-known/openid-configuration`)).json();
const jwks = createRemoteJWKSet(new URL(oidc.jwks_uri));

export async function verify(token) {
  const { payload } = await jwtVerify(token, jwks, {
    issuer: oidc.issuer,
    audience: CLIENT_ID, // or your RFC 8707 resource URI for resource-bound tokens
  });
  return payload;
}
```

Always fetch `/.well-known/jwks.json` (don't pin a copy): `kid` is the key's RFC 7638
thumbprint and changes when the key rotates. Offline verification cannot see logout or
session revocation; use the SDK path when that matters.

## Optional: role and permission gates

- `validateJWTToken` / `ValidateJWTTokenRequest` accept `roles` to require roles.
- For per-object permissions, call `check_permissions` with the caller's token — see
  `authorizer-fga`.

## Verify

```bash
# 1. A valid token is accepted by the server-side check
curl -s -X POST "$AUTHORIZER_URL/v1/validate_jwt_token" \
  -H 'Content-Type: application/json' -H "Origin: $AUTHORIZER_URL" \
  -d "{\"token_type\":\"access_token\",\"token\":\"$USER_TOKEN\"}" | jq .is_valid   # true

# 2. Your API: 401 without a token, 200 with one
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3001/protected                       # 401
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $USER_TOKEN" http://localhost:3001/protected   # 200

# 3. A tampered token is rejected (expect 401)
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer ${USER_TOKEN}x" http://localhost:3001/protected

# 4. JWKS path only: at least one key is published
curl -s "$AUTHORIZER_URL/.well-known/jwks.json" | jq '.keys | length'   # >= 1
```

(`USER_TOKEN` comes from `authorizer-local-setup` step 4; adjust the port to your API.)

Docs: https://docs.authorizer.dev/core/oauth2-oidc · https://docs.authorizer.dev/sdks/authorizer-go ·
https://docs.authorizer.dev/sdks/authorizer-js/functions · https://docs.authorizer.dev/sdks/authorizer-python
