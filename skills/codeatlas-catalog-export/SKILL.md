---
name: codeatlas-catalog-export
description: "Use when you need the whole CodeAtlas service catalog as a file (jsonl, csv or md) via the asynchronous export API, or need several services in one request with the batch lookup."
---

# CodeAtlas catalog export and batch lookup

Two ways to get many services at once:

| Need | Use | Auth |
| --- | --- | --- |
| A few named services in one call (1 to 50 slugs) | `POST /api/v1/services/batch` | none |
| The whole catalog as a downloadable file | `POST /api/v1/exports` then poll then download | Bearer `api:write` |

For a plain paged read, `GET /api/v1/services?limit=100&cursor=...` is simpler than an export; the catalog is small. Use exports when you want a file in `csv` or `md`.

## Batch lookup (public)

```bash
curl -s -X POST https://codeatlas.com.br/api/v1/services/batch \
  -H 'Content-Type: application/json' \
  -d '{"slugs":["zelvo-ai","consultoria","nope"]}'
```

```json
{"count":3,"results":[
 {"slug":"zelvo-ai","status":200,"data":{"slug":"zelvo-ai","name":"...","capabilities":["..."],"deliveryTimeline":"...","url":"..."}},
 {"slug":"consultoria","status":200,"data":{"...":"..."}},
 {"slug":"nope","status":404,"error":{"code":"not_found","message":"No CodeAtlas service matches that slug.","hint":"Use one of the slugs returned by GET /api/v1/services."}}]}
```

- The HTTP status is `200` even when some items fail. Read each item's `status` (`200`, `404` or `400`) and `error`.
- `slugs` must be an array of 1 to 50 strings. Empty, longer than 50, or not an array returns `422 unprocessable` with the hint `Send an array of 1 to 50 service slugs.`
- Valid production slugs: `produto-digital`, `zelvo-ai`, `consultoria`.
- Limit: 60 per minute per IP (`RateLimit-Policy: "batch";q=60;w=60`).
- Sandbox: `https://codeatlas.com.br/api/sandbox/v1/services/batch` with `sandbox-one` and `sandbox-two`.

## Asynchronous export

Flow: `POST /exports` returns `202` and a `Location` URL, you poll it, then you download.

### 1. Get a token

See `codeatlas-agent-auth`. Use the sandbox audience while testing:

```bash
SANDBOX=https://codeatlas.com.br/api/sandbox/v1
TOKEN=$(curl -s -u "$CLIENT_ID:$CLIENT_SECRET" -d grant_type=client_credentials \
  -d resource=$SANDBOX https://codeatlas.com.br/oauth2/token | jq -r .access_token)
```

### 2. Start the job

`format` is required: `jsonl`, `csv` or `md`. No other field is allowed. Send an `Idempotency-Key` (8 to 128 printable ASCII): the same key and body within 24 hours returns the same job with `Idempotency-Replayed: true` instead of creating a new one.

```bash
curl -s -i -X POST $SANDBOX/exports \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -H "Idempotency-Key: $(uuidgen)" -d '{"format":"csv"}'
```

```http
HTTP/2 202
location: https://codeatlas.com.br/api/sandbox/v1/exports/exp_EyFs5ZncLOeIEik3dLrxrQ
retry-after: 1
ratelimit-policy: "export";q=10;w=3600

{"data":{"id":"exp_EyFs5ZncLOeIEik3dLrxrQ","status":"queued","format":"csv","download_url":null,
 "created_at":"2026-10-14T11:13:53.132Z","expires_at":"2026-10-15T11:13:53.132Z"}}
```

Production: the same at `https://codeatlas.com.br/api/v1/exports` with a production token.

### 3. Poll

`GET` the `Location` URL. `status` moves `queued`, `running`, `succeeded`. Honor `Retry-After` (seconds) while it is not finished and keep a deadline (about 60 seconds is plenty; in the sandbox the first poll was already `succeeded`).

```bash
curl -s $SANDBOX/exports/$ID -H "Authorization: Bearer $TOKEN"
```

```json
{"data":{"id":"exp_...","status":"succeeded","format":"csv",
 "download_url":"https://codeatlas.com.br/api/sandbox/v1/exports/exp_.../download","...":"..."}}
```

`download_url` is `null` until `succeeded`.

### 4. Download

The download needs the same Bearer token.

```bash
curl -s -o catalog.csv "$DOWNLOAD_URL" -H "Authorization: Bearer $TOKEN"
```

| Format | Content-Type | Shape |
| --- | --- | --- |
| `csv` | `text/csv; charset=utf-8` | header `slug,name,service_type,capabilities,delivery_timeline,url,description`, capabilities joined with `; ` |
| `jsonl` | `application/jsonl; charset=utf-8` | one service JSON object per line |
| `md` | `text/markdown; charset=utf-8` | `# CodeAtlas services` then one `##` section per service |

The response has `Content-Disposition: attachment; filename="codeatlas-services.<ext>"`.

Rules:

- Jobs and files are kept for 24 hours (`expires_at`). Download once and store it.
- Jobs are private to the OAuth client that created them. Another client (or a wrong or expired id) gets `404 not_found`.
- Limit: 10 exports per hour per OAuth client (`429` with `Retry-After` after that). Reuse one export instead of creating several.
- The file is the same public catalog you can read without a token. Do not use exports to bypass the read rate limit.

### Complete script

```bash
set -euo pipefail
SANDBOX=https://codeatlas.com.br/api/sandbox/v1
LOC=$(curl -s -D - -o /dev/null -X POST $SANDBOX/exports \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -H "Idempotency-Key: $(uuidgen)" -d '{"format":"jsonl"}' | awk 'tolower($1)=="location:"{print $2}' | tr -d '\r')
for _ in $(seq 1 30); do
  R=$(curl -s "$LOC" -H "Authorization: Bearer $TOKEN")
  [ "$(echo "$R" | jq -r .data.status)" = succeeded ] && break
  sleep 2
done
curl -s -o catalog.jsonl "$(echo "$R" | jq -r .data.download_url)" -H "Authorization: Bearer $TOKEN"
```

## CLI and SDKs

The CLI, SDKs and install steps are in `codeatlas-cli-sdk`. All of them create the job, wait for `succeeded` and download.

```bash
codeatlas services batch sandbox-one nope --sandbox            # public
codeatlas export csv --out catalog.csv --sandbox               # needs CODEATLAS_CLIENT_ID / _SECRET
```

```ts
const results = await client.services.batch(["zelvo-ai", "consultoria"]); // items with status 404 are not thrown
const job = await client.exports.create("jsonl");
await client.exports.wait(job.id);
const file = await client.exports.download(job.id);
```

```python
results = client.services.batch(["zelvo-ai", "consultoria"])
job = client.exports.wait(client.exports.create("jsonl").id)
file = client.exports.download(job.id)
```

```go
results, _ := client.Services.Batch(ctx, []string{"zelvo-ai", "consultoria"})
job, _ := client.Exports.Create(ctx, codeatlas.ExportJSONL)
client.Exports.Wait(ctx, job.ID, nil) // polls every 1s, gives up after 60s
file, _ := client.Exports.Download(ctx, job.ID) // []byte
```

## Errors

| Status and code | Cause and fix |
| --- | --- |
| `401 unauthorized` | No Bearer token on `/exports`. See `codeatlas-agent-auth`. |
| `401 invalid_token` | Expired, revoked or wrong audience. A production token on the sandbox (or the reverse) fails with a hint naming the right `resource`. |
| `404 not_found` on `/exports/{id}` or `/download` | Wrong id, expired (24 hours), or created by another client. |
| `422 unprocessable` | `format` missing or not `jsonl`, `csv`, `md` (`Use one of: jsonl, csv, md.`), unknown field, or `Idempotency-Key` reused with a different body. |
| `400 bad_request` | Malformed JSON or invalid `Idempotency-Key`. |
| `415 unsupported_media_type` | Send `Content-Type: application/json`. |
| `429 rate_limited` | 10 exports per hour per client (60 per minute for batch, 120 per minute for reads). Wait `Retry-After` seconds. |

Error shape: `{"error":{"code","message","hint"}}`.
