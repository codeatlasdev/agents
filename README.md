# CodeAtlas for agents

[CodeAtlas](https://codeatlas.com.br) is a technology consultancy in Curitiba, Paraná, Brazil. It maps operational problems before deciding between process, design, automation, data or software. This repository packages what CodeAtlas exposes to AI agents.

## What is exposed

Everything is public, read-only and needs no credentials. There is no write path: an agent cannot hire CodeAtlas or send data through these endpoints.

| Surface | Where |
| --- | --- |
| REST API | `https://codeatlas.com.br/api/v1` ([OpenAPI](https://codeatlas.com.br/openapi.json)) |
| MCP server (Streamable HTTP) | `https://codeatlas.com.br/mcp`, tools `codeatlas_list_services`, `codeatlas_get_service`, `codeatlas_search_services` |
| MCP docs server | `https://codeatlas.com.br/mcp/docs`, tools `codeatlas_get_documentation`, `codeatlas_get_auth_info` |
| A2A agent | `https://codeatlas.com.br/.well-known/agent-card.json` (`POST /a2a`) |
| NLWeb | `https://codeatlas.com.br/ask` |
| llms.txt | `https://codeatlas.com.br/llms.txt` |
| MCP Registry | `br.com.codeatlas/mcp` on [registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io/v0/servers?search=codeatlas) |

The catalog has three services: `produto-digital`, `zelvo-ai` and `consultoria`. CodeAtlas publishes no price list; scope, price and timeline are defined after a diagnostic conversation (contato@codeatlas.com.br).

## What is in this repository

This repository is an [Agent Plugins](https://agent-plugins.org/specification) 1.0.0 package.

- `plugin.json`: manifest.
- `mcp.json`: the two remote MCP servers (Agent Plugins format).
- `.mcp.json`: the same servers in the project-level format used by Claude Code and similar clients.
- `skills/codeatlas-agent-integration`: how to discover and safely call the API, MCP and A2A endpoints.
- `skills/codeatlas-service-discovery`: how to match an operational challenge to a CodeAtlas service.
- `AGENTS.md`: rules for coding agents that integrate with CodeAtlas.

The skills are also served at `https://codeatlas.com.br/.well-known/agent-skills/index.json`. The website is the source of truth; if the two differ, trust the website.

## Use the MCP server

```bash
claude mcp add --transport http codeatlas https://codeatlas.com.br/mcp
```

## License

MIT. See [LICENSE](LICENSE).
