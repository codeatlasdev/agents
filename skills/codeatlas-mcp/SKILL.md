---
name: codeatlas-mcp
description: "Use when connecting an MCP client or agent to the CodeAtlas MCP server (https://codeatlas.com.br/mcp, Streamable HTTP) to list, get or search services, or to submit a contact request with a Bearer token."
---

# Connect to the CodeAtlas MCP server

Two remote servers, both Streamable HTTP, JSON-RPC 2.0 over `POST`:

| Server | URL | Tools |
| --- | --- | --- |
| CodeAtlas | `https://codeatlas.com.br/mcp` | `codeatlas_list_services`, `codeatlas_get_service`, `codeatlas_search_services`, `codeatlas_request_contact` |
| CodeAtlas docs | `https://codeatlas.com.br/mcp/docs` | `codeatlas_get_documentation`, `codeatlas_get_auth_info` |

Registry name: `br.com.codeatlas/mcp` (<https://registry.modelcontextprotocol.io/v0/servers?search=codeatlas>). Server card: <https://codeatlas.com.br/mcp/server-card>.

## Connect

```bash
# Claude Code
claude mcp add --transport http codeatlas https://codeatlas.com.br/mcp
claude mcp add --transport http codeatlas-docs https://codeatlas.com.br/mcp/docs

# With a token for codeatlas_request_contact
claude mcp add --transport http codeatlas https://codeatlas.com.br/mcp \
  --header "Authorization: Bearer $CODEATLAS_TOKEN"
```

Other clients: `.mcp.json` (project level) or `mcp.json` (Agent Plugins) in <https://github.com/codeatlasdev/agents>:

```json
{"mcpServers":{"codeatlas":{"type":"http","url":"https://codeatlas.com.br/mcp"}}}
```

The Agent Plugins file uses `"type": "streamable-http"`.

## Which tools need auth

| Tool | Auth | Parameters |
| --- | --- | --- |
| `codeatlas_list_services` | none | optional `limit` (1 to 100), `cursor` |
| `codeatlas_get_service` | none | required `slug`: `produto-digital`, `zelvo-ai` or `consultoria` |
| `codeatlas_search_services` | none | required `query` (business challenge or capability, Portuguese or English) |
| `codeatlas_request_contact` | **Bearer `api:write`, production audience** | required `name`, `email`, `message`; optional `company`, `locale` (`pt-BR`, `en`), `service_slug`, `idempotency_key` |

The first three are `readOnlyHint: true`. `codeatlas_request_contact` has `readOnlyHint: false` and `openWorldHint: true`: it sends a real email. Follow the consent rules in `codeatlas-contact-request` before calling it.

There is no sandbox MCP endpoint and a sandbox token is rejected here. To rehearse the write, use the REST sandbox (`codeatlas-sandbox-testing`).

## Raw protocol

The server is stateless: `initialize` returns no `Mcp-Session-Id`, `GET /mcp` returns `405` (no server-initiated SSE stream), `ping` answers `{}`, notifications answer `202`. Send both content types so strict clients are happy.

```bash
H=(-H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream')

curl -s -X POST https://codeatlas.com.br/mcp "${H[@]}" -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"my-agent","version":"1"}}}'
# {"id":1,"jsonrpc":"2.0","result":{"capabilities":{"tools":{"listChanged":false}},"protocolVersion":"2025-06-18",
#   "serverInfo":{"name":"br.com.codeatlas/mcp","title":"CodeAtlas","version":"1.0.0"},"instructions":"..."}}

curl -s -X POST https://codeatlas.com.br/mcp "${H[@]}" -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'

curl -s -X POST https://codeatlas.com.br/mcp "${H[@]}" -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"codeatlas_get_service","arguments":{"slug":"zelvo-ai"}}}'
```

A tool result is `{"result":{"content":[{"type":"text","text":"<JSON string>"}]}}`. Parse `content[0].text` as JSON: `{"data":{...}}` for `get_service`, `{"count":N,"data":[...],"next_cursor":...}` for list, `{"count":N,"data":[...]}` for search.

Authenticated write (production token, no `resource` or `resource=https://codeatlas.com.br/api/v1`; get it with `codeatlas-agent-auth`):

```bash
curl -s -X POST https://codeatlas.com.br/mcp "${H[@]}" -H "Authorization: Bearer $TOKEN" -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"codeatlas_request_contact","arguments":{"name":"Maria Souza","email":"maria@example.com","message":"Quero conversar sobre um produto digital sob medida.","service_slug":"produto-digital","idempotency_key":"6f1c1c52-8d0e-4c1e-9a53-3a4d6a0f9d11"}}}'
```

Pass an `idempotency_key` so a retry never emails twice. The REST endpoint documents a limit of 5 contact requests per hour per OAuth client; the MCP responses carry no `RateLimit` headers, so budget the same limit and do not retry on your own.

Docs server tools: `codeatlas_get_documentation` takes optional `topic` (`api_catalog`, `auth`, `docs`, `llms_txt`, `openapi`) and returns links; `codeatlas_get_auth_info` takes optional `format` (`json` or `text`) and describes the auth model (read: none, write: OAuth `client_credentials`).

## Errors to expect

Auth failures are HTTP errors, not tool errors. Tool validation failures are HTTP `200` with `isError: true`. Protocol failures are JSON-RPC `error` objects.

| Case | Response |
| --- | --- |
| `codeatlas_request_contact` without token | HTTP `401`, `WWW-Authenticate: Bearer resource_metadata="https://codeatlas.com.br/.well-known/oauth-protected-resource", scope="api:write"`, body `{"error":{"code":"unauthorized",...}}`. Get a token (`codeatlas-agent-auth`) and send it in the `Authorization` header of the HTTP request. |
| Sandbox-audience (or expired or revoked) token | HTTP `401 invalid_token`, hint `Request one with resource=https://codeatlas.com.br/api/v1.` |
| Bad tool arguments (token present) | HTTP `200`, `{"result":{"isError":true,"content":[{"type":"text","text":"{\"error\":\"Field \\\"name\\\" is invalid. Use between 2 and 120 characters.\"}"}]}}`. Missing ones: `Missing required argument: name, email, message.` |
| `get_service` with a bad slug | HTTP `200`, `isError: true`, `Argument slug must be one of: produto-digital, zelvo-ai, consultoria.` |
| Unknown tool | JSON-RPC error `-32602`, `Unknown tool: <name>.` |
| Unknown method | JSON-RPC error `-32601`, `Method not found: <method>.` |
| `GET /mcp` | HTTP `405`, `The MCP endpoint accepts POST.` |
| Rate limited | The MCP read path sends no `RateLimit` headers. If you receive HTTP `429`, wait `Retry-After` seconds. |

Read tools work with or without a token; sending a valid token to them is harmless.

Related: `codeatlas-agent-integration` (REST and A2A), `codeatlas-service-discovery` (matching a problem to a service).
