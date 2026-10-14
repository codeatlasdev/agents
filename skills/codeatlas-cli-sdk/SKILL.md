---
name: codeatlas-cli-sdk
description: "Use when you want the codeatlas CLI or the JavaScript, Python or Go SDK to call the CodeAtlas API instead of hand-written HTTP: how to install each, call it, and handle errors."
---

# CodeAtlas CLI and SDKs

All four clients talk to `https://codeatlas.com.br/api/v1`, are MIT licensed and have the same surface: list, get, search (client-side filter, the API has no search endpoint), batch, health, register, contact requests, exports. Reads need no auth; contact requests and exports need OAuth client credentials (see `codeatlas-agent-auth`). Every write client supports the sandbox (`codeatlas-sandbox-testing`).

## Install (state on 2026-10-14)

| Client | Works today | Not available yet |
| --- | --- | --- |
| CLI | `brew install codeatlasdev/tap/codeatlas` (formula 0.1.0, prebuilt macOS and Linux binaries), or `npx -y github:codeatlasdev/codeatlas-sdk-js <command>` | `npx @codeatlas/sdk` (package not on npm) |
| JavaScript / TypeScript, package `@codeatlas/sdk` | `npm install github:codeatlasdev/codeatlas-sdk-js` or `bun add --trust github:codeatlasdev/codeatlas-sdk-js` | `npm install @codeatlas/sdk` returns 404 |
| Python, package `codeatlas-sdk`, import `codeatlas` | `pip install git+https://github.com/codeatlasdev/codeatlas-sdk-python` (or `uv pip install git+https://...`) | `pip install codeatlas-sdk` returns 404 |
| Go, module `github.com/codeatlasdev/codeatlas-sdk-go` | `go get github.com/codeatlasdev/codeatlas-sdk-go@v0.1.0` (Go 1.23 or newer, standard library only) | nothing missing |

Traps:

- **Do not run `pip install codeatlas`.** That PyPI name belongs to an unrelated project ("A Python library for visualizing and understanding codebases"). The package name here is `codeatlas-sdk`, installed from Git until it is published.
- The JavaScript package builds on install (`prepare` runs `tsc`). Bun skips lifecycle scripts for Git dependencies unless you pass `--trust`; without it `dist/` is missing and imports fail. npm 11 may print an `allow-scripts` warning but still builds.
- Registries may start serving these names later. Re-check with `npm view @codeatlas/sdk version` and `pip index versions codeatlas-sdk` before assuming a Git install is still needed. The website docs (<https://codeatlas.com.br/docs>) are the source of truth.

## CLI

```text
codeatlas services list [--limit N] [--cursor C] [--json]
codeatlas services get <slug> [--json]
codeatlas services search <query> [--json]
codeatlas services batch <slug> [<slug>...] [--json]
codeatlas health [--json]
codeatlas export <jsonl|csv|md> [--out FILE] [--json]            needs credentials
codeatlas register <client-name> [--json]
codeatlas contact --name N --email E --message M [--company C] [--service SLUG]
                  [--locale pt-BR|en] [--idempotency-key K] [--sandbox] [--json]
Options: --json, --base-url U (or $CODEATLAS_API_URL), --sandbox, -h, -v
```

```bash
codeatlas services list --json
codeatlas services get zelvo-ai
codeatlas services batch zelvo-ai consultoria nope        # unknown slugs are 404 items, not a crash
codeatlas register my-agent                                # prints the CODEATLAS_CLIENT_ID / _SECRET exports; shown once
export CODEATLAS_CLIENT_ID=ca_... CODEATLAS_CLIENT_SECRET=cas_...
codeatlas export csv --out catalog.csv --sandbox           # creates the job, waits, downloads
codeatlas contact --sandbox --name "Maria Souza" --email maria@example.com \
  --message "Quero conversar sobre um produto digital sob medida." --service produto-digital \
  --idempotency-key "$(uuidgen)"                           # cr_... received (sandbox)
```

Without `--sandbox`, `register`, `export` and `contact` hit production: `contact` then emails the CodeAtlas team. Follow `codeatlas-contact-request` first. Errors print as `error <status> <code>: <message>` and `hint: ...`, for example `error 404 not_found: No sandbox service matches that slug.`

## JavaScript / TypeScript

```ts
import { CodeAtlas, CodeAtlasError, SANDBOX_BASE_URL } from "@codeatlas/sdk";

const reader = new CodeAtlas();                                   // reads: no credentials
const page = await reader.services.list({ limit: 20 });           // page.count, page.data, page.next_cursor
for await (const s of reader.services.iterate()) console.log(s.slug);
const svc = await reader.services.get("zelvo-ai");
const hits = await reader.services.search("whatsapp");            // client-side filter
const items = await reader.services.batch(["zelvo-ai", "nope"]);  // items[i].status 200 or 404

const { clientId, clientSecret } = await CodeAtlas.register("my-agent"); // real registration; store the secret
const client = new CodeAtlas({ baseUrl: SANDBOX_BASE_URL, credentials: { clientId, clientSecret } });
const r = await client.contactRequests.create(
  { name: "Maria Souza", email: "maria@example.com", message: "Quero conversar sobre um produto digital." },
  { idempotencyKey: crypto.randomUUID() },
);                                                                 // r.id, r.status ("received"), r.delivery ("sandbox")
const job = await client.exports.create("jsonl");                  // "jsonl" | "csv" | "md"
await client.exports.wait(job.id);
const file = await client.exports.download(job.id);

try { await reader.services.get("nope"); }
catch (e) { if (e instanceof CodeAtlasError) console.log(e.status, e.code, e.hint); }
```

## Python

```python
import uuid
from codeatlas import CodeAtlas, CodeAtlasError, Credentials, SANDBOX_BASE_URL

reader = CodeAtlas()
page = reader.services.list(limit=20)
for s in reader.services.iterate(): print(s.slug)
svc = reader.services.get("zelvo-ai")
results = reader.services.batch(["zelvo-ai", "nope"])     # [r.status for r in results] -> [200, 404]

creds = CodeAtlas.register("my-agent")                    # real registration: Credentials(client_id, client_secret)
client = CodeAtlas(base_url=SANDBOX_BASE_URL, credentials=creds)
r = client.contact_requests.create(
    "Maria Souza", "maria@example.com", "Quero conversar sobre um produto digital.",
    service_slug="produto-digital", idempotency_key=str(uuid.uuid4()),
)                                                          # r.id, r.status, r.delivery == "sandbox"
job = client.exports.wait(client.exports.create("jsonl").id)
file = client.exports.download(job.id)

try: reader.services.get("nope")
except CodeAtlasError as e: print(e.status, e.code, e.hint)
```

Standard library only; the import name is `codeatlas`.

## Go

```go
import (
	"context"
	"errors"
	"fmt"

	codeatlas "github.com/codeatlasdev/codeatlas-sdk-go"
)

ctx := context.Background()
reader := codeatlas.New()
page, _ := reader.Services.List(ctx, &codeatlas.ListParams{Limit: 20})
svc, _ := reader.Services.Get(ctx, "zelvo-ai")
res, _ := reader.Services.Batch(ctx, []string{"zelvo-ai", "nope"}) // res[1].Status == 404
for s, err := range reader.Services.All(ctx, nil) { /* follows next_cursor */ }

creds, _ := codeatlas.Register(ctx, "my-agent") // real registration
client := codeatlas.New(codeatlas.WithBaseURL(codeatlas.SandboxBaseURL), codeatlas.WithCredentials(creds.ClientID, creds.ClientSecret))
receipt, err := client.ContactRequests.Create(ctx, codeatlas.ContactRequestInput{
	Name: "Maria Souza", Email: "maria@example.com", Message: "Quero conversar sobre um produto digital.",
}, codeatlas.WithIdempotencyKey("6f1c1c52-8d0e-4c1e-9a53-3a4d6a0f9d11"))
job, _ := client.Exports.Create(ctx, codeatlas.ExportJSONL) // ExportJSONL | ExportCSV | ExportMD
client.Exports.Wait(ctx, job.ID, nil)                        // polls every 1s, gives up after 60s
file, _ := client.Exports.Download(ctx, job.ID)              // []byte

var apiErr *codeatlas.APIError
if _, err := reader.Services.Get(ctx, "nope"); errors.As(err, &apiErr) {
	fmt.Println(apiErr.Status, apiErr.Code, apiErr.Hint, codeatlas.IsNotFound(err), codeatlas.IsRateLimited(err))
}
fmt.Println(reader.LastRateLimit()) // parsed RateLimit headers of the latest call
```

Options: `WithBaseURL`, `WithCredentials`, `WithHTTPClient` (default timeout 15s), `WithOrigin` (tests only). The Go client keeps the latest `RateLimit` values and puts them on the error; `apiErr.RetryAfter` has the `Retry-After` wait.

## Behaviors common to all clients

- The client requests and caches the OAuth token itself, with `resource` set to its base URL, so one set of credentials serves sandbox and production. Tokens last 3600 seconds.
- Errors carry `status`, `code`, `message`, `hint` from the API error (`{"error":{"code","message","hint"}}`). OAuth errors (`invalid_client`) use the same type.
- Rate limits: reads 120 per minute per IP, batch 60 per minute, contact requests 5 per hour per client, exports 10 per hour per client, `register` 10 per hour per IP. On `429` wait `Retry-After`; do not loop.
- Always pass an idempotency key to production `contactRequests.create` so a retry cannot send twice.
- Never log or commit `client_secret`. `register` returns it once.

Source: <https://github.com/codeatlasdev/codeatlas-sdk-js>, <https://github.com/codeatlasdev/codeatlas-sdk-python>, <https://github.com/codeatlasdev/codeatlas-sdk-go>. API docs: <https://codeatlas.com.br/docs>.
