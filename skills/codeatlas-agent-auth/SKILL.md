---
name: codeatlas-agent-auth
description: "Use when an agent must call a CodeAtlas write endpoint (contact requests, exports, the MCP tool codeatlas_request_contact) and needs a Bearer token: register an OAuth client, get a token, check identity, revoke."
---

# CodeAtlas agent authentication

Reads (`GET /api/v1/*`, `POST /api/v1/services/batch`, the three read-only MCP tools) are public. Only these need a token:

- `POST /api/v1/contact-requests`
- `POST /api/v1/exports`, `GET /api/v1/exports/{id}`, `GET /api/v1/exports/{id}/download`
- MCP tool `codeatlas_request_contact`

Authority: <https://codeatlas.com.br/auth.md>. Hosts: the API and the authorization server are both `https://codeatlas.com.br`. One scope exists: `api:write`.

## Which method works today

| Method | Status in production (checked 2026-10-14) |
| --- | --- |
| Dynamic client registration + `client_credentials` | Works. Use this. |
| `service_auth` (claim ceremony with a person's email) | Implemented, but `POST /agent/identity` answers `503 temporarily_unavailable` because the confirmation email cannot be sent (no mail sender configured). Do not rely on it. |
| `anonymous`, `identity_assertion` | Not offered. They answer `anonymous_not_enabled` / `identity_assertion_not_enabled`. |

## Steps

1. **Register once.** `client_name` is 2 to 80 characters, recorded and attached to every contact request, so name the agent honestly. The secret is returned once and only its hash is stored: save it in your secret store. Limit: 10 registrations per hour per IP.

   ```bash
   curl -s -X POST https://codeatlas.com.br/oauth2/register \
     -H 'Content-Type: application/json' \
     -d '{"client_name":"acme-procurement-agent"}'
   ```

   ```json
   {"client_id":"ca_...","client_secret":"cas_...","client_secret_expires_at":0,
    "grant_types":["client_credentials"],"scope":"api:write",
    "token_endpoint_auth_method":"client_secret_basic","client_name":"acme-procurement-agent",
    "client_id_issued_at":1791976365}
   ```

2. **Get a token.** Pick the audience with `resource`. Use the sandbox audience while testing (see the `codeatlas-sandbox-testing` skill). Sandbox and production tokens are not interchangeable. Limit: 30 token requests per minute per IP.

   ```bash
   # sandbox
   curl -s -u "$CLIENT_ID:$CLIENT_SECRET" \
     -d grant_type=client_credentials -d scope=api:write \
     -d resource=https://codeatlas.com.br/api/sandbox/v1 \
     https://codeatlas.com.br/oauth2/token
   # production: omit resource (default) or pass resource=https://codeatlas.com.br/api/v1
   ```

   ```json
   {"access_token":"<JWT>","token_type":"Bearer","expires_in":3600,"scope":"api:write"}
   ```

   `client_id` / `client_secret` in the form body also work instead of HTTP Basic. There is no refresh token: after 3600 seconds repeat this step.

3. **Check who you are.** `GET /agent/identity` accepts both audiences.

   ```bash
   curl -s https://codeatlas.com.br/agent/identity -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   {"audience":"https://codeatlas.com.br/api/sandbox/v1","client_id":"ca_...","email":null,
    "expires_at":"...","issued_at":"...","issuer":"https://codeatlas.com.br",
    "registration_type":"dynamic_client","sandbox":true,"scopes":["api:write"]}
   ```

   `email` is `null` for dynamic clients.

4. **Revoke** when done or if the token leaks (RFC 7009). It answers `200` with an empty body and is idempotent. You can only revoke tokens of your own client. After revocation the token is rejected immediately with `401 invalid_token`.

   ```bash
   curl -s -i -u "$CLIENT_ID:$CLIENT_SECRET" \
     -d "token=$TOKEN" -d token_type_hint=access_token \
     https://codeatlas.com.br/oauth2/revoke
   ```

   There is no agent-callable way to delete a client. Ask <contato@codeatlas.com.br> with the `client_id`.

## service_auth claim flow (when email is available)

Meant for acting on behalf of a person whose email you have. Documented in `auth.md`; today it is blocked by the mail sender (see table above).

1. `POST https://codeatlas.com.br/agent/identity` with `{"type":"service_auth","login_hint":"person@example.com"}`. Limit: 10 per hour per IP.
2. On success you get `claim_token` (shown once), `claim.user_code` (6 digits), `claim.verification_uri`, `claim.expires_in` (600) and `claim.interval` (5). CodeAtlas emails the person a link to `verification_uri`. The email does not contain the code.
3. Show the person `verification_uri` and `user_code`. They type the code on that page, not back to you. Five wrong codes lock the attempt.
4. Poll `POST https://codeatlas.com.br/oauth2/token` with `grant_type=urn:workos:agent-auth:grant-type:claim&claim_token=clm_...` no faster than every 5 seconds. `authorization_pending` means wait, `slow_down` means add 5 seconds, `expired_token` means restart, `invalid_grant` means unknown or expired `claim_token`. Success returns the same token response as step 2 with the confirmed email in the token.

What production returns today for step 1:

```http
HTTP/2 503
Retry-After: 60

{"error":"temporarily_unavailable","error_description":"The confirmation email could not be sent. Retry later, or use POST https://codeatlas.com.br/oauth2/register."}
```

No registration is usable after a `503`. Fall back to dynamic client registration. Re-check by retrying the request after `Retry-After`; do not loop.

## SDK and CLI

```bash
codeatlas register acme-procurement-agent          # prints CODEATLAS_CLIENT_ID / CODEATLAS_CLIENT_SECRET exports
```

```ts
import { CodeAtlas } from "@codeatlas/sdk";
const { clientId, clientSecret } = await CodeAtlas.register("acme-procurement-agent");
const client = new CodeAtlas({ credentials: { clientId, clientSecret } }); // fetches and caches the token
```

```python
from codeatlas import CodeAtlas
creds = CodeAtlas.register("acme-procurement-agent")   # Credentials(client_id, client_secret)
client = CodeAtlas(credentials=creds)
```

```go
creds, err := codeatlas.Register(ctx, "acme-procurement-agent")
client := codeatlas.New(codeatlas.WithCredentials(creds.ClientID, creds.ClientSecret))
```

Install the SDKs with the `codeatlas-cli-sdk` skill. Each `Register` call is a real registration against production and counts toward the 10 per hour per IP limit.

## Errors

API errors: `{"error":{"code","message","hint"}}`. `/oauth2/*` and `/agent/identity` errors: `{"error","error_description"}`.

| Response | Cause and fix |
| --- | --- |
| `401 unauthorized` with `WWW-Authenticate: Bearer resource_metadata=...` | No token. Do steps 1 and 2. |
| `401 invalid_token` | Expired, revoked, malformed, or wrong audience (the hint says which `resource` to use). Request a new token with the right `resource`. |
| `401 invalid_client` | Unknown client or wrong secret. Register again. |
| `400 invalid_target` | `resource` must be `https://codeatlas.com.br/api/v1` or `https://codeatlas.com.br/api/sandbox/v1`. |
| `400 invalid_scope` | Only `api:write` exists. |
| `400 unsupported_grant_type` | Only `client_credentials` and `urn:workos:agent-auth:grant-type:claim`. |
| `400 invalid_client_metadata` | `client_name` missing or not 2 to 80 characters. |
| `400 anonymous_not_enabled`, `400 identity_assertion_not_enabled` | `/agent/identity` only takes `service_auth`. |
| `400 invalid_request` | `service_auth` without `login_hint`. |
| `503 temporarily_unavailable` | `service_auth` mail sender unavailable. Use dynamic registration. |
| `429 rate_limited` | Wait `Retry-After` seconds. `RateLimit` and `RateLimit-Policy` headers show the quota. |

Never print or log `client_secret` or tokens. Never commit them.
