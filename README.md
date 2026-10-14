# CodeAtlas for agents

[CodeAtlas](https://codeatlas.com.br) is a technology consultancy in Curitiba, Paraná, Brazil. It maps operational problems before deciding between process, design, automation, data or software. This repository packages what CodeAtlas exposes to AI agents.

## Install the skills

```bash
npx skills add codeatlasdev/agents
```

List them first without installing:

```bash
npx skills add codeatlasdev/agents --list
```

Pick one with `--skill`, for example `npx skills add codeatlasdev/agents --skill codeatlas-contact-request`.

## Skills by capability

| Area | Skill | Use it to |
| --- | --- | --- |
| Discover | [`codeatlas-agent-integration`](skills/codeatlas-agent-integration/SKILL.md) | Find CodeAtlas data: llms.txt, OpenAPI, the read-only REST API, MCP and A2A. |
| Discover | [`codeatlas-service-discovery`](skills/codeatlas-service-discovery/SKILL.md) | Match an operational problem to `produto-digital`, `zelvo-ai` or `consultoria`. |
| Authenticate | [`codeatlas-agent-auth`](skills/codeatlas-agent-auth/SKILL.md) | Register an OAuth client, get a `client_credentials` token, check identity, revoke. Explains the `service_auth` claim flow and its current status. |
| Act | [`codeatlas-contact-request`](skills/codeatlas-contact-request/SKILL.md) | Submit a first-conversation request for a person: consent, `Idempotency-Key`, rate limits. |
| Act | [`codeatlas-catalog-export`](skills/codeatlas-catalog-export/SKILL.md) | Export the catalog (`202`, poll, download) and batch-look-up up to 50 services. |
| Connect | [`codeatlas-mcp`](skills/codeatlas-mcp/SKILL.md) | Connect to `https://codeatlas.com.br/mcp`: tools and which ones need a token. |
| Build | [`codeatlas-cli-sdk`](skills/codeatlas-cli-sdk/SKILL.md) | Install and use the `codeatlas` CLI and the JavaScript, Python and Go SDKs. |
| Build | [`codeatlas-sandbox-testing`](skills/codeatlas-sandbox-testing/SKILL.md) | Exercise reads, auth, contact requests and exports safely in `/api/sandbox/v1`. |

Reads are public and need no credentials. Writes (contact requests, exports, the MCP tool `codeatlas_request_contact`) need an OAuth Bearer token. Test them in the sandbox first: nothing is emailed or stored there.

## What is exposed

| Surface | Where |
| --- | --- |
| REST API | `https://codeatlas.com.br/api/v1` ([OpenAPI](https://codeatlas.com.br/openapi.json)) |
| Sandbox | `https://codeatlas.com.br/api/sandbox/v1` (fixture data, nothing delivered) |
| OAuth (write endpoints) | `https://codeatlas.com.br/oauth2/register`, `/oauth2/token`, `/oauth2/revoke`; [auth.md](https://codeatlas.com.br/auth.md) |
| MCP server (Streamable HTTP) | `https://codeatlas.com.br/mcp`, tools `codeatlas_list_services`, `codeatlas_get_service`, `codeatlas_search_services`, `codeatlas_request_contact` (Bearer token) |
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
- `skills/`: the eight skills above.
- `AGENTS.md`: rules for coding agents that integrate with CodeAtlas.

The skills are also served at `https://codeatlas.com.br/.well-known/agent-skills/index.json`. The website is the source of truth; if the two differ, trust the website.

## Use the MCP server

```bash
claude mcp add --transport http codeatlas https://codeatlas.com.br/mcp
```

## License

MIT. See [LICENSE](LICENSE).
