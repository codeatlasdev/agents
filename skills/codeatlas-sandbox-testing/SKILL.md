---
name: codeatlas-sandbox-testing
description: "Use when building or testing a CodeAtlas integration and you must exercise reads, batch, auth, contact requests and exports without emailing the team or touching production data."
---

# Test CodeAtlas integrations in the sandbox

`https://codeatlas.com.br/api/sandbox/v1` mirrors the whole API with fixture data. It is part of the production origin, not a separate host. Always run a write flow here before you run it against `https://codeatlas.com.br/api/v1`.

What it guarantees:

- Contact requests are validated and accepted with `delivery: "sandbox"`. Nothing is emailed and nothing is stored.
- Exports and batch work on the fixtures.
- Same paths, same validation, same rate-limit headers, same error shapes as production.

What it does not do:

- It is not a separate account system. `POST /oauth2/register` has no sandbox: registering creates a real client (10 per hour per IP). The same client works in both environments; only the token audience differs.
- Data is not production data. Slugs are `sandbox-one` and `sandbox-two` only. Production slugs return `404` here, and `service_slug` in a contact request must still be a production slug (`produto-digital`, `zelvo-ai`, `consultoria`).
- Rate limits are real: the contact limit (5 per hour per client) and export limit (10 per hour per client) also apply to sandbox calls. Do not loop.
- `/mcp` has no sandbox endpoint. `POST /api/sandbox/mcp` and `/mcp/sandbox` answer `405`.

## Quick start

```bash
B=https://codeatlas.com.br
S=$B/api/sandbox/v1

# 0. Index and health, no auth
curl -s $B/api/sandbox
curl -s $S/health        # {"sandbox":true,"service":"codeatlas-api","status":"ok","version":"1.0.0"}

# 1. Reads, no auth
curl -s $S/services
curl -s $S/services/sandbox-one
curl -s -X POST $S/services/batch -H 'Content-Type: application/json' -d '{"slugs":["sandbox-one","nope"]}'

# 2. Register once (a real client; keep the secret out of logs)
curl -s -X POST $B/oauth2/register -H 'Content-Type: application/json' -d '{"client_name":"my-integration-tests"}'

# 3. Token with the sandbox audience
TOKEN=$(curl -s -u "$CLIENT_ID:$CLIENT_SECRET" -d grant_type=client_credentials \
  -d resource=$S $B/oauth2/token | jq -r .access_token)

# 4. Who am I? sandbox:true confirms the audience
curl -s $B/agent/identity -H "Authorization: Bearer $TOKEN" | jq '{audience,sandbox,client_id}'

# 5. Contact request
curl -s -X POST $S/contact-requests -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -H "Idempotency-Key: $(uuidgen)" \
  -d '{"name":"Maria Souza","email":"maria@example.com","message":"Quero conversar sobre um produto digital sob medida.","service_slug":"produto-digital"}'

# 6. Export: POST, poll the Location header, download (see codeatlas-catalog-export)
curl -s -i -X POST $S/exports -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -H "Idempotency-Key: $(uuidgen)" -d '{"format":"csv"}'
```

Observed results (2026-10-14):

| Call | Result |
| --- | --- |
| `GET $S/services` | `200`, `{"count":2,"data":[sandbox-one, sandbox-two],"next_cursor":null,"sandbox":true}`. `limit` and `cursor` are accepted but the two fixtures always come back together. |
| `GET $S/services/produto-digital` | `404 not_found`, hint `Use sandbox-one or sandbox-two.` |
| `POST $S/contact-requests` | `201`, `{"data":{"id":"cr_...","status":"received","delivery":"sandbox","created_at":"..."}}`, `RateLimit-Policy: "contact";q=5;w=3600` |
| Same call, same `Idempotency-Key` | `201` with `Idempotency-Replayed: true` and the same `cr_` id |
| Same key, different body | `422 unprocessable` |
| `POST $S/exports` | `202`, `Location: $S/exports/exp_...`, `Retry-After: 1`; the first poll was already `succeeded` |
| Production-audience token on `$S` | `401 invalid_token`, hint `Request one with resource=https://codeatlas.com.br/api/sandbox/v1.` |
| Sandbox-audience token on `https://codeatlas.com.br/api/v1` or `/mcp` | `401 invalid_token`, hint names `resource=https://codeatlas.com.br/api/v1` |
| `POST $S/exports` without token | `401`, `WWW-Authenticate` points to `https://codeatlas.com.br/.well-known/oauth-protected-resource/api/sandbox/v1` |

## Negative tests worth running

Each of these should fail exactly like this; they cost nothing but quota.

```bash
# 422: validation names the field and the rule
curl -s -X POST $S/contact-requests -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"name":"M","email":"nope","message":"oi"}'
# {"error":{"code":"unprocessable","message":"Field \"name\" is invalid.","hint":"Use between 2 and 120 characters."}}

# 422: unknown field
#   {"error":{"code":"unprocessable","message":"Unknown field(s): extra.","hint":"Allowed fields: company, email, locale, message, name, service_slug."}}
# 422: sandbox slug as service_slug
#   {"error":{"code":"unprocessable","message":"Field \"service_slug\" is invalid.","hint":"Use one of: produto-digital, zelvo-ai, consultoria."}}
# 415: Content-Type text/plain  -> unsupported_media_type
# 413: body over 16384 bytes    -> payload_too_large
# 400: Idempotency-Key "abc"    -> bad_request, hint "Use 8 to 128 printable ASCII characters, for example a UUID."
# 405: GET /contact-requests    -> method_not_allowed, hint "Use POST with a JSON body."
# 429: sixth contact request in an hour -> rate_limited with Retry-After
```

## Point your SDK or CLI at the sandbox

```bash
codeatlas health --sandbox
codeatlas services list --sandbox
codeatlas services batch sandbox-one nope --sandbox
CODEATLAS_CLIENT_ID=ca_... CODEATLAS_CLIENT_SECRET=cas_... codeatlas export csv --sandbox --out out.csv
CODEATLAS_CLIENT_ID=ca_... CODEATLAS_CLIENT_SECRET=cas_... codeatlas contact --sandbox \
  --name "Maria Souza" --email maria@example.com --message "Quero conversar sobre um produto digital." --service produto-digital
```

```ts
import { CodeAtlas, SANDBOX_BASE_URL } from "@codeatlas/sdk";
const client = new CodeAtlas({ baseUrl: SANDBOX_BASE_URL, credentials: { clientId, clientSecret } });
```

```python
from codeatlas import CodeAtlas, Credentials, SANDBOX_BASE_URL
client = CodeAtlas(base_url=SANDBOX_BASE_URL, credentials=Credentials(client_id, client_secret))
```

```go
client := codeatlas.New(codeatlas.WithBaseURL(codeatlas.SandboxBaseURL), codeatlas.WithCredentials(id, secret))
```

The SDKs request the token with `resource` set to their base URL, so the same credentials work for both environments. Switching to production is a one-line change (drop the base URL); until then no real email can leave.

## Going to production

1. Sandbox flow passes, including a replayed `Idempotency-Key` and a `429` handler that honors `Retry-After`.
2. The person you act for has confirmed the exact contact-request payload (`codeatlas-contact-request`).
3. Request a token without `resource` (or `resource=https://codeatlas.com.br/api/v1`), call `https://codeatlas.com.br/api/v1/...`. Expect `delivery: "delivered"` (or `stored`) instead of `sandbox`.
4. Never use production write endpoints for tests. The CodeAtlas team reads those emails.

Related: `codeatlas-agent-auth`, `codeatlas-contact-request`, `codeatlas-catalog-export`. Reference: <https://codeatlas.com.br/auth.md>, <https://codeatlas.com.br/openapi.json> (`x-sandbox`).
