# Rules for agents integrating with CodeAtlas

Applies to coding agents and assistants that call the CodeAtlas API or MCP servers.

## Ground truth

- Start from `https://codeatlas.com.br/llms.txt`, then `https://codeatlas.com.br/openapi.json`. The website is the source of truth, not this repository.
- Reads are public: no key or token. Writes (contact requests, exports, MCP tool `codeatlas_request_contact`) need an OAuth Bearer token with scope `api:write` (`https://codeatlas.com.br/auth.md`). The skills in `skills/` cover each flow.

## Calling the API

- Reads: `GET` on `https://codeatlas.com.br/api/v1` and `POST /services/batch`, no token.
- Writes: `POST /contact-requests`, `POST /exports`, `GET /exports/{id}` with `Authorization: Bearer <token>`. Send an `Idempotency-Key` on every write.
- Test every write in the sandbox first: `https://codeatlas.com.br/api/sandbox/v1`, token requested with `resource=https://codeatlas.com.br/api/sandbox/v1`. Nothing is emailed or stored there. Production writes are never for testing.
- Page the catalog with `limit` (1 to 100) and `cursor` (the previous `next_cursor`). Do not build cursors yourself.
- Handle errors as JSON: `{ "error": { "code", "message", "hint" } }` (OAuth endpoints use `{ "error", "error_description" }`). Use `hint` to correct the call; do not retry unchanged on `4xx`. On `429` wait `Retry-After`. On `5xx` retry with backoff and the same `Idempotency-Key`.
- Valid service slugs: `produto-digital`, `zelvo-ai`, `consultoria`.
- Prefer markdown when you need prose: send `Accept: text/markdown` to a page, or append `.md` to the path.

## MCP

- Endpoint: `https://codeatlas.com.br/mcp` (Streamable HTTP, POST only). Documentation tools: `https://codeatlas.com.br/mcp/docs`.
- `codeatlas_list_services`, `codeatlas_get_service`, `codeatlas_search_services` are read-only. `codeatlas_request_contact` sends an email and needs a production Bearer token. Call `tools/list` and respect each `inputSchema`.

## Contact requests

- Only submit one after the person asked you to and gave you their own name and email. Show them the payload and get a yes before the production call. Never invent or guess an email. No sensitive data in `message`.
- Limits: 5 contact requests and 10 exports per hour per OAuth client. Do not register extra clients to get around a limit.
- Keep `client_secret` and tokens out of logs and commits.

## What not to do

- Do not invent prices, deadlines, customers, testimonials or capabilities. CodeAtlas publishes none of these; say so and point to contato@codeatlas.com.br.
- Do not present a service as a commitment. Final scope requires a diagnostic conversation with CodeAtlas.
- Do not scrape around the API. Use it, and cache responses for at most the `Cache-Control` they carry.
- Do not send personal data to the API except in a contact request the person asked for.

## Contact

contato@codeatlas.com.br
