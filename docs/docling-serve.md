# docling-serve — direct API reference

Document conversion push API on hades.home. Verified live 2026-11-15 against
docling-serve v1.38.2 (image `quay.io/docling-project/docling-serve-cpu:v1.38.2`).

## Endpoints and auth

| What | Value |
| --- | --- |
| Base URL (LAN) | `http://hades.home:5001` |
| Base URL (from hades containers) | `http://docling-serve:5001` |
| Auth | Header `X-Api-Key: <key>` on all convert/chunk endpoints |
| Key location | `hades:/opt/docling-serve/.env` (line `DOCLING_SERVE_API_KEY=...`) |
| Machine-readable spec | `http://hades.home:5001/openapi.json` (23 endpoints) |
| Interactive docs | `http://hades.home:5001/docs` (Swagger), `/scalar` |

`/health` and the docs/spec pages need no key. There is no cookie or
query-parameter auth alternative — header only. The key was generated on the
host and is never printed in logs or memory; read it from the `.env` file.

```bash
KEY=$(ssh xdblue@hades.home 'sudo grep "^DOCLING_SERVE_API_KEY=" /opt/docling-serve/.env | cut -d= -f2-')
```

## Convert a local file (multipart)

```bash
curl -s -H "X-Api-Key: $KEY" \
  -F "files=@document.pdf" \
  -F "to_formats=md" \
  http://hades.home:5001/v1/convert/file
```

## Convert from JSON (URL or base64 push)

```bash
curl -s -H "X-Api-Key: $KEY" -H "Content-Type: application/json" \
  -d '{
    "sources": [
      { "kind": "http", "url": "https://example.com/doc.pdf" }
    ],
    "options": { "to_formats": ["md"] }
  }' \
  http://hades.home:5001/v1/convert/source
```

Push file bytes with `kind: "file"`:

```json
{ "kind": "file", "base64_string": "<base64 of file>", "filename": "document.pdf" }
```

**The body key is `sources`** — a unified array discriminated by `kind`
(`http` | `file`). The legacy `file_sources` / `http_sources` shape shown in
some doc pages is rejected with 422 on v1.38.2.

## Response shape

```json
{
  "document": { "md_content": "## ...", "html_content": null, "...": "one field per requested to_format" },
  "status": 200,
  "errors": [],
  "processing_time": 4.2,
  "timings": { },
  "confidence": null
}
```

`to_formats`: `md`, `json` (full DoclingDocument), `yaml`, `html`, `text`,
`doctags`, `chunks`, `latex`.

Common `options`: `do_ocr` (default true), `ocr_engine` (`auto` default),
`pdf_backend` (`docling_parse` default), `table_mode` (`accurate` default),
`images_scale` (default 2), `image_export_mode`
(`placeholder` | `embedded` | `referenced`), `page_range`, `abort_on_error`.

## Long documents — async flow

```
POST /v1/convert/source/async   (same JSON body)  -> {"task_id": "...", "task_status": "pending"}
GET  /v1/status/poll/{task_id}                    -> task_status: pending | started | success | failure
GET  /v1/result/{task_id}                         -> the convert response, once success
```

(Endpoint paths unchanged by the host-port move — only the LAN port changed
from 5002 to 5001 on 2026-11-16, user request.)

Also available: `POST /v1/convert/file/async`, batch `POST /v1/convert/source/batch`,
chunking endpoints (`/v1/chunk/hierarchical/*`, `/v1/chunk/hybrid/*`),
WebSocket progress at `/v1/status/ws/{task_id}`.

## Gotchas

- Set generous client timeouts. OCR-heavy PDFs can run minutes; a dropped
  connection does **not** cancel the conversion — it finishes server-side and
  wastes CPU (protocol cancellation verified non-aborting, 2026-11-15).
- Health endpoint `/health` is exempt from auth; everything under `/v1/` is not
  (401 without the key).
- Related services (as of 2026-11-16): the docling-mcp MCP container and its
  MCPHub entries were **removed** — this REST API is the only docling service.
  If an MCP front is ever re-added, the wiring notes (X-Api-Key header name,
  timeout, NO_PROXY, `body` argument wrapping, ~170K-token tool schemas) are
  preserved in `memory/servers/hades.home/memory.md`.

## Deployment facts (hades.home)

- Compose: `/opt/docling-serve/docker-compose.yml`; API key in sibling `.env`
  (mode 0600). Same image as the removed docling MCP container, default CMD
  (binds `0.0.0.0:5001` in-container), host publish `5001:5001` — the
  framework-default port, chosen at the user's explicit request on 2026-11-16
  (port verified free before binding; earlier deployment used 5002 per the
  default-port+1 convention). `mem_limit: 6g`, restart unless-stopped, on the
  `hades-lan` docker network.
- No host firewall is active on hades (ufw/firewalld inactive, INPUT ACCEPT) —
  the API key is the only gate on the LAN-reachable port.
- Rollback: `cd /opt/docling-serve && sudo docker compose down` then remove the
  directory. The image is now used only by this container (the docling MCP
  container was removed 2026-11-16), so `docker rmi
  quay.io/docling-project/docling-serve-cpu:v1.38.2` is safe as part of a full
  removal. Full changelog: `memory/servers/hades.home/changelog.log`
  (2026-11-15/16 entries).
