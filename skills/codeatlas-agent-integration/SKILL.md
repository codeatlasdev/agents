---
name: codeatlas-agent-integration
description: "Use when an agent needs CodeAtlas data: call the read-only REST API or MCP server to list, get or search services."
---

# CodeAtlas agent integration

Use this skill when an agent needs CodeAtlas service information or integration metadata.

1. Fetch `https://codeatlas.com.br/llms.txt` and `https://codeatlas.com.br/docs`.
2. Fetch `https://codeatlas.com.br/openapi.json` before calling the API.
3. Prefer `GET https://codeatlas.com.br/api/v1/services` for the complete catalog. It accepts `limit` and `cursor`.
4. Use `GET /api/v1/services/{slug}` for one service. Valid slugs are `produto-digital`, `zelvo-ai`, and `consultoria`.
5. The API is public and read-only. No API key or token is needed.
6. Do not invent prices or deadlines. CodeAtlas scopes work after a diagnostic conversation; contact `contato@codeatlas.com.br`.
7. For natural-language matching, POST `{"query":"..."}` to `/ask`.
8. For MCP clients, use `https://codeatlas.com.br/mcp` (Streamable HTTP). Documentation tools are at `https://codeatlas.com.br/mcp/docs`.
9. For A2A clients, read `/.well-known/agent-card.json` and send `message/send` to `/a2a`.

Errors are JSON and include `error.code`, `error.message`, and `error.hint`.
