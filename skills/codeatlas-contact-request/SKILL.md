---
name: codeatlas-contact-request
description: "Use when a person asks you to contact CodeAtlas for a first conversation (diagnostic, quote, partnership) and you must submit it through the API, CLI, SDK or MCP tool on their behalf, safely and without duplicates."
---

# Submit a CodeAtlas contact request

`POST /api/v1/contact-requests` emails the CodeAtlas team a first-conversation request on behalf of a person. CodeAtlas answers that person at the email you give. It sends a real email in their name, so treat it as an irreversible action.

Needs a Bearer token with scope `api:write`. Get one with the `codeatlas-agent-auth` skill. Rehearse in the sandbox first (`codeatlas-sandbox-testing`).

## Consent requirements

The API has no `consent` field and does not verify anything about the person. The obligation is yours:

1. The person must have asked you to contact CodeAtlas, in this conversation.
2. The person must have given you their own name and email. Never invent, guess, scrape or reuse an email or name from elsewhere.
3. Show the person what you will send (name, email, company, message, service) and get a yes before the production call.
4. The email you submit is where CodeAtlas replies. It must be one the person reads.
5. Do not include sensitive data (documents, health, credentials, financial data) in `message`. Describe the problem in a few sentences; the diagnostic conversation collects the rest.
6. Send one request per intent. Do not retry with a new `Idempotency-Key` after a success.

Pricing and deadlines are not quoted by this API; CodeAtlas defines scope after the conversation. Do not promise any.

## Request

Headers: `Authorization: Bearer <token>`, `Content-Type: application/json`, `Idempotency-Key: <8 to 128 printable ASCII>` (use a UUID, generated once per intent and reused on retries).

| Field | Rules |
| --- | --- |
| `name` | required, 2 to 120 characters, the person's full name |
| `email` | required, valid email, up to 320 characters |
| `message` | required, 10 to 2000 characters |
| `company` | optional, 1 to 160 characters |
| `locale` | optional, `pt-BR` (default) or `en`, the reply language |
| `service_slug` | optional, one of `produto-digital`, `zelvo-ai`, `consultoria` |

Send only the fields above: an unknown field returns `422` (`Allowed fields: company, email, locale, message, name, service_slug.`). The body must be a JSON object, `application/json`, under 16384 bytes.

```bash
# Sandbox first. TOKEN was requested with resource=https://codeatlas.com.br/api/sandbox/v1
KEY=$(uuidgen)
curl -s -i -X POST https://codeatlas.com.br/api/sandbox/v1/contact-requests \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -H "Idempotency-Key: $KEY" \
  -d '{"name":"Maria Souza","email":"maria@example.com","company":"Acme","message":"Quero conversar sobre um produto digital sob medida.","locale":"pt-BR","service_slug":"produto-digital"}'
```

Response `201`:

```json
{"data":{"id":"cr_KHIPwLRjkCKYHSSP","status":"received","delivery":"sandbox","created_at":"2026-10-14T11:13:14.716Z"}}
```

For production use the same call at `https://codeatlas.com.br/api/v1/contact-requests` with a production token (no `resource`). `data.delivery` then is:

- `delivered`: the email reached the CodeAtlas team.
- `stored`: saved for the team, email delivery was unavailable. Still a success; do not resend.
- `sandbox`: sandbox only, nothing sent or stored.

## Idempotency

The same key and the same body within 24 hours replays the first response with `Idempotency-Replayed: true` and sends no second email. The same key with a different body returns `422`. A key shorter than 8 characters returns `400`. The key is optional, but always send it on production calls so that a retry after a timeout cannot email twice.

```bash
# repeat the exact call with the same $KEY:
# HTTP/2 201, idempotency-replayed: true, same cr_ id
```

## Rate limits

5 contact requests per hour per OAuth client. Every response carries `RateLimit-Policy: "contact";q=5;w=3600` and `RateLimit: "contact";r=<remaining>;t=<seconds to reset>` (plus legacy `X-RateLimit-*`). The sixth call returns `429` with `Retry-After` in seconds (about an hour). Replays of an idempotency key do not need a new request, so keep and reuse the key. Do not register extra clients to get around the limit. Counters are eventually consistent, so a burst can slightly exceed it.

## MCP, CLI and SDK

MCP (needs the same Bearer header on the HTTP request to `https://codeatlas.com.br/mcp`, production audience):

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"codeatlas_request_contact","arguments":{"name":"Maria Souza","email":"maria@example.com","message":"Quero conversar sobre um produto digital sob medida.","service_slug":"produto-digital","idempotency_key":"6f1c1c52-8d0e-4c1e-9a53-3a4d6a0f9d11"}}}
```

CLI (`--sandbox` validates without sending; it prints `cr_... received (sandbox)`):

```bash
export CODEATLAS_CLIENT_ID=ca_... CODEATLAS_CLIENT_SECRET=cas_...
codeatlas contact --name "Maria Souza" --email maria@example.com \
  --message "Quero conversar sobre um produto digital sob medida." \
  --service produto-digital --idempotency-key "$(uuidgen)" --sandbox
```

```ts
import { CodeAtlas, SANDBOX_BASE_URL } from "@codeatlas/sdk";
const client = new CodeAtlas({ baseUrl: SANDBOX_BASE_URL, credentials: { clientId, clientSecret } });
const r = await client.contactRequests.create(
  { name: "Maria Souza", email: "maria@example.com", message: "Quero conversar sobre um produto digital sob medida." },
  { idempotencyKey: crypto.randomUUID() },
);
console.log(r.id, r.status, r.delivery); // cr_..., received, sandbox
```

```python
from codeatlas import CodeAtlas, Credentials, SANDBOX_BASE_URL
client = CodeAtlas(base_url=SANDBOX_BASE_URL, credentials=Credentials(client_id, client_secret))
r = client.contact_requests.create("Maria Souza", "maria@example.com",
    "Quero conversar sobre um produto digital sob medida.", idempotency_key="6f1c1c52-8d0e-4c1e-9a53-3a4d6a0f9d11")
```

```go
client := codeatlas.New(codeatlas.WithBaseURL(codeatlas.SandboxBaseURL), codeatlas.WithCredentials(id, secret))
r, err := client.ContactRequests.Create(ctx, codeatlas.ContactRequestInput{
	Name: "Maria Souza", Email: "maria@example.com", Message: "Quero conversar sobre um produto digital sob medida.",
}, codeatlas.WithIdempotencyKey("6f1c1c52-8d0e-4c1e-9a53-3a4d6a0f9d11"))
```

Install instructions: `codeatlas-cli-sdk`.

## Errors

| Status and code | Cause and fix |
| --- | --- |
| `401 unauthorized` | No Bearer token. See `codeatlas-agent-auth`. |
| `401 invalid_token` | Expired, revoked, or wrong audience (a sandbox token on production or the reverse). Request a token with the right `resource`. |
| `403 forbidden` | Token lacks `api:write`. Documented in the OpenAPI; not reproducible today because `api:write` is the only scope a token can have. |
| `400 bad_request` | Invalid `Idempotency-Key` (8 to 128 printable ASCII) or a body that is not a valid JSON object. |
| `415 unsupported_media_type` | Send `Content-Type: application/json`. |
| `413 payload_too_large` | Body over 16384 bytes. Shorten `message` (limit 2000 characters). |
| `422 unprocessable` | A field failed validation; `error.message` names it and `error.hint` gives the rule (for example `Use between 2 and 120 characters.`, or `Use one of: produto-digital, zelvo-ai, consultoria.`). Also returned for a reused `Idempotency-Key` with a different body. Fix the payload; do not retry unchanged. |
| `405 method_not_allowed` | Only `POST`. |
| `429 rate_limited` | Quota of 5 per hour used. Wait `Retry-After` seconds. |
| `5xx` | Retry with exponential backoff and the same `Idempotency-Key`. |

Error shape: `{"error":{"code","message","hint"}}`.

After a successful production call, tell the person the request id (`cr_...`), the `delivery` value, and that CodeAtlas will reply to their email. You cannot check the status afterwards: there is no read endpoint for contact requests. Direct contact: <contato@codeatlas.com.br>.
