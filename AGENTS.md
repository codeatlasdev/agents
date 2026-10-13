# Rules for agents integrating with CodeAtlas

Applies to coding agents and assistants that call the public CodeAtlas API or MCP servers.

## Ground truth

- Start from `https://codeatlas.com.br/llms.txt`, then `https://codeatlas.com.br/openapi.json`. The website is the source of truth, not this repository.
- The API and MCP servers are public and read-only. Do not look for API keys, OAuth or tokens: there are none (`https://codeatlas.com.br/auth.md`).

## Calling the API

- Use only `GET` (and `HEAD`/`OPTIONS`) on `https://codeatlas.com.br/api/v1`. Other methods return `405`.
- Page the catalog with `limit` (1 to 100) and `cursor` (the previous `next_cursor`). Do not build cursors yourself.
- Handle errors as JSON: `{ "error": { "code", "message", "hint" } }`. Codes: `bad_request`, `not_found`, `method_not_allowed`, `unsupported_media_type`. Use `hint` to correct the call; do not retry unchanged on `4xx`.
- Valid service slugs: `produto-digital`, `zelvo-ai`, `consultoria`.
- Prefer markdown when you need prose: send `Accept: text/markdown` to a page, or append `.md` to the path.

## MCP

- Endpoint: `https://codeatlas.com.br/mcp` (Streamable HTTP, POST only). Documentation tools: `https://codeatlas.com.br/mcp/docs`.
- Every tool is read-only and idempotent. Call `tools/list` and respect each `inputSchema`.

## What not to do

- Do not invent prices, deadlines, customers, testimonials or capabilities. CodeAtlas publishes none of these; say so and point to contato@codeatlas.com.br.
- Do not present a service as a commitment. Final scope requires a diagnostic conversation with CodeAtlas.
- Do not scrape around the API. Use it, and cache responses for at most the `Cache-Control` they carry.
- Do not send personal data to these endpoints. They do not need it and do not store it.

## Contact

contato@codeatlas.com.br
