---
name: notebooklm-mcp
description: Deploy and maintain the notebooklm-py MCP server (hades, Docker) and re-auth its Google session. TRIGGERS - notebooklm, notebooklm-py, notebooklm login, master token, re-auth, refresh cookies, port 9420, MCPHub notebooklm.
allowed-tools: Read, Edit, Bash
---

# NotebookLM MCP (notebooklm-py on hades)

> **Self-Evolving Skill**: If instructions are wrong, flags drifted, or a workaround
> was needed — fix this file immediately, don't defer. Only for real, reproducible issues.

## When to Use This Skill

- NotebookLM tools through MCPHub return AUTH errors (Google session died)
- Re-running the one-time `notebooklm login --master-token` bootstrap
- Deploying the stack from scratch on a new host
- Adding a second Google account (one container per account — see "Multi-account")
- Verifying the deployment after a reboot or container recreate

## Architecture

| Component          | Location                                             | Notes                                            |
| ------------------ | ---------------------------------------------------- | ------------------------------------------------ |
| MCP server         | hades `/opt/notebooklm` (compose)                    | image `tenglin/notebooklm-mcp:0.8.6` (pinned)    |
| Container          | `notebooklm-notebooklm-mcp-1`, uid 1001             | healthcheck built in                             |
| Listen             | `0.0.0.0:9420` on hades                             | bearer-gated (401 anon)                          |
| Bearer token       | `/opt/notebooklm/.env` (`NOTEBOOKLM_MCP_TOKEN`, 0600) | server-generated, never leaves the host          |
| Google profile     | hades `/opt/notebooklm/profiles/server` ← container `/data/profiles/server` (rw) | contains `master_token.json` + `storage_state.json`, both 0600, self-contained incl. android_id |
| Egress             | gluetun KR proxy `http://hades.home:8010`            | **required** — see "Region gate" below           |
| TLS front (Caddy)   | hades `/opt/caddy`, publishes 9443 → container 9420  | `tls internal`, CN `hades.home` — see "File transfer" |
| File transfer       | `NOTEBOOKLM_MCP_PUBLIC_URL=https://hades.home:9443`   | enables studio_download / uploads — required, bare https origin only |
| MCPHub upstream    | name `notebooklm`, streamable-http `http://hades.home:9420/mcp` | owner `admin` (SSRF guard), visibility public, Bearer header |
| Hub route          | `http://hades.home:3001/mcp/notebooklm`              | pi mcp.json entry `notebooklm` (deferred)        |
| Bootstrap tooling  | workstation: `~/.local/bin/notebooklm` (uv tool, 0.8.6) | browser must run HERE, not on hades              |

Account: dedicated Gmail account (`scroll.notebook@gmail.com`).

## 1. Health Check (read-only, start here)

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 xdblue@hades.home '
docker compose -f /opt/notebooklm/docker-compose.yml ps --format "{{.Name}} {{.Status}}"
docker logs notebooklm-notebooklm-mcp-1 2>&1 | tail -3'
```

- `healthy` + a 200 line for `notebook.google.com` in the logs = auth alive
- `Master-token re-mint failed` or `served a signed-out page` = auth dead → section 2
- Full end-to-end probe (from workstation, proves hub → server → Google):

```bash
H="Authorization: Bearer $(cat ~/.config/mcphub/pi.token)"
A="Accept: application/json, text/event-stream"   # REQUIRED: hub rejects without it (406)
U=http://hades.home:3001/mcp/notebooklm
curl -sS -D /tmp/nlm-hdr -o /dev/null -X POST -H "$H" -H "$A" -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' $U
SID=$(grep -i '^mcp-session-id:' /tmp/nlm-hdr | tr -d '\r' | cut -d' ' -f2)
curl -sS -o /dev/null -X POST -H "$H" -H "$A" -H "mcp-session-id: $SID" -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}' $U
curl -sS -X POST -H "$H" -H "$A" -H "mcp-session-id: $SID" -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"notebooklm-notebook_list","arguments":{"limit":5}}}' $U
```

`"isError":false` = deployment fully healthy. Empty `notebooks` list is normal (dedicated account).

## 2. Re-Auth (the recurring task)

The master token normally re-mints cookies automatically (self-healing). Re-auth is only
needed if the master token itself is revoked (logs: `perform_oauth rejected the master token`).
Requires a human: Google sign-in happens in a real browser on the workstation.

**Rate-limit rule: at most ONE bootstrap attempt per day per account.** If Google starts
rejecting (`This browser or app may not be secure`, rate-limited sign-in, token revoked
within minutes), STOP and wait ~24h. Rapid retries deepen the lockout (verified the hard way,
2026-11-09). Before retrying, sign into the account once in a normal browser to give it
organic history.

### Steps (workstation, ~2 min plus human sign-in)

1. Kill any stale CDP Chrome and start a fresh one (proxy + debug port):

```bash
PID=$(ss -tlnp 2>/dev/null | awk '$4 ~ /:9222$/ {print $NF}' | grep -oP 'pid=\K[0-9]+' | head -1)
[ -n "$PID" ] && kill $PID
rm -rf /tmp/nlm-cdp-profile
nohup google-chrome-stable --user-data-dir=/tmp/nlm-cdp-profile --remote-debugging-port=9222 \
  --proxy-server="http://hades.home:8010" --no-first-run --no-default-browser-check about:blank \
  >/tmp/nlm-chrome.log 2>&1 & disown
sleep 4 && curl -sS http://localhost:9222/json/version | head -c 80
```

**Do NOT kill this Chrome with `pkill -f`** — the pattern matches the invoking shell's own
command line and kills the script (exit 143, twice verified). Kill by the `ss` PID as above.

2. Run the bootstrap (human signs in with the account in the Chrome window; complete any
   Google security challenge inside that window):

```bash
HTTP_PROXY=http://hades.home:8010 HTTPS_PROXY=http://hades.home:8010 NO_PROXY=localhost,127.0.0.1 \
  ~/.local/bin/notebooklm login --master-token --account scroll.notebook@gmail.com --cdp-url http://localhost:9222
```

Success prints `Master-token login OK`.

3. Verify the re-mint works BEFORE shipping (this is the part that dies when Google is unhappy):

```bash
HTTP_PROXY=http://hades.home:8010 HTTPS_PROXY=http://hades.home:8010 NO_PROXY=localhost,127.0.0.1 \
  ~/.local/bin/notebooklm auth refresh
```

`ok refreshed` = token healthy. `perform_oauth rejected` = STOP, see troubleshooting.

4. Ship profile to hades and restart (server reloads the profile only at process start):

```bash
rsync -a --exclude='*.lock' -e "ssh -o BatchMode=yes -o ConnectTimeout=5" \
  ~/.notebooklm/profiles/default/ xdblue@hades.home:/opt/notebooklm/profiles/server/
ssh -o BatchMode=yes -o ConnectTimeout=5 xdblue@hades.home \
  'cd /opt/notebooklm && docker compose restart notebooklm-mcp && sleep 6 && docker logs notebooklm-notebooklm-mcp-1 2>&1 | tail -2'
```

5. Run the end-to-end probe from section 1.

6. Clean up the CDP Chrome (by PID from `ss`, never `pkill -f`):

```bash
rm -rf /tmp/nlm-cdp-profile
```

**Single-writer rule**: after step 4, hades is the ONLY active session consumer. Never run
`notebooklm auth refresh` against the local `~/.notebooklm` profile afterwards — two active
sessions from one account supersede each other's cookies. The local copy is bootstrap-only.

## 3. Fresh Deployment (condensed)

Prereqs: Docker host, gluetun proxy reachable, MCPHub running, a browser machine for bootstrap.

1. Compose dir `/opt/notebooklm`: image `tenglin/notebooklm-mcp:0.8.6`, uid
   `1001:1001`, profile dir mounted rw at `/data/profiles/server`,
   `NOTEBOOKLM_PROFILE=server`, `.env` 0600 with `NOTEBOOKLM_MCP_TOKEN=<openssl rand -hex 32>`,
   `NOTEBOOKLM_MCP_ALLOW_EXTERNAL_BIND: "1"`, ports `9420:9420`, and the egress env (below).
2. Compose `environment:` MUST include the proxy (region gate, below):

```yaml
      HTTP_PROXY: http://hades.home:8010
      HTTPS_PROXY: http://hades.home:8010
      NO_PROXY: localhost,127.0.0.1
```

3. `docker compose up -d`, then verify gate: anonymous `curl http://hades:9420/mcp` → 401;
   with `Authorization: Bearer <token>` + initialize → 200.
4. Bootstrap the Google profile on the browser machine (section 2, steps 1–3), ship it (step 4).
5. Register in MCPHub via admin API (JWT from `POST /api/auth/login`, header `X-Auth-Token`):

```json
{"name":"notebooklm","config":{"type":"streamable-http","url":"http://<host>:9420/mcp",
 "headers":{"Authorization":"Bearer <token>"},"owner":"admin","visibility":"public"}}
```

   `owner: admin` is required for private-IP upstreams (SSRF guard). Do NOT set
   `NOTEBOOKLM_AUTH_JSON` on the server — it breaks multi-profile serving.
6. Verify: MCPHub shows `connected` with 38 tools (`notebooklm-*`), then run the section 1 probe.
7. pi client: add deferred `notebooklm` entry to `~/.pi/agent/mcp.json` pointing at the hub
   route with the `!echo Bearer $(cat $HOME/.config/mcphub/pi.token)` header (see existing entries).

## 4. Multi-account (not yet deployed)

One container per account, each with its own profile dir, bearer token, and MCPHub server —
the route name picks the account (no per-call `profile` parameter). Alternative: single
container with `NOTEBOOKLM_MCP_PROFILES=a,b` — but then EVERY tool call must pass `profile`.

## 5. File Transfer (studio_download / uploads) — Caddy TLS front

MCP tool responses are JSON only; files (PDF/audio/PNG) can't ride the MCP channel. The
server instead mints a signed `https://…/files/dl/{token}` URL (HMAC, token = sole auth on
that route) and the client fetches it out-of-band. `NOTEBOOKLM_MCP_PUBLIC_URL` is the origin
baked into those links. The server NEVER fetches this URL itself — it's config, not a proxy.

**Hard constraint:** the value must be a BARE https origin (`mcp/_urlcheck.py` validator;
port allowed, no path/userinfo/query). `http://…` → container exits at startup. Since the
container serves plain http on 9420, an https terminator in front is mandatory → Caddy.

### Current setup (hades, verified 2026-11-10)

- `/opt/caddy`: Caddy 2.11.9 (pinned), `tls internal` (Caddy Local Authority ECC root, 10y,
  CN `hades.home`), publishes 9443, `reverse_proxy notebooklm-notebooklm-mcp-1:9420` over
  the external network `notebooklm_default`. `admin off`, `auto_https disable_redirects`.
- Env: `NOTEBOOKLM_MCP_PUBLIC_URL=https://hades.home:9443` in `/opt/notebooklm/.env` **AND**
  in the compose `environment:` map — `env_file` vars not in the `environment:` map are a
  SILENT NO-OP (compose enumerates explicitly; verified the hard way). Backups:
  `.env.bak-publicurl`, `docker-compose.yml.bak-publicurl`.
- Workstation trust: Caddy root CA at `~/.local/share/ca-certificates-extra/caddy-local-root.crt`;
  system anchor `/etc/ca-certificates/trust-source/anchors/` (Arch path — NOT
  `/usr/local/share/ca-certificates/`, that's Debian), then `sudo trust extract-compat`.
- Verify end-to-end: `curl -sS -o /dev/null -w '%{http_code}\n' https://hades.home:9443/mcp -X POST
  -H 'Content-Type: application/json' -d '{}'` → 401 = TLS good + bearer gate good.

### Gotchas

- Signing key is ephemeral per container restart: outstanding links die with a restart;
  new calls mint fresh links. Nothing to fix, just don't cache links long-term.
- The same gate also blocks file uploads (`source_add` type=file → `await_upload`) and
  inline text bodies for reports/data-tables — one env var fixes all three.
- Caddy root CA rotates if `/data` volume is wiped → re-export and re-trust the CA.
- EVERY client machine fetching artifacts needs the CA installed (athena done; others
  pending — see "Caddy TLS on hades" in `memory/network.md` for the per-OS paths).

### Fresh-deploy steps (condensed)

1. Compose at `/opt/caddy` (image `caddy:2.11.9`, join the notebooklm container's network
   as external), Caddyfile: `https://<host>:9443 { tls internal; reverse_proxy <container>:9420 }`.
2. `NOTEBOOKLM_MCP_PUBLIC_URL=https://<host>:9443` into `.env` AND compose `environment:` map,
   then `docker compose up -d --force-recreate` (a plain `up -d` won't pick up env changes).
3. Export the root CA: `docker exec caddy cat /data/caddy/pki/authorities/local/root.crt`
   → trust it on every client machine that will fetch links.
4. Verify: anonymous https POST to `:9443/mcp` → 401; then a real `studio_download`.

## Region gate (why the proxy env is mandatory)

The home IP (Hong Kong) is region-unsupported by NotebookLM: `notebook.google.com` 302s to an
"unsupported location" page, which the server reports as `Authentication expired or invalid …
served a signed-out page`. Both gluetun exits tested OK (KR 8010 and US 8011 return 200).
The container's httpx/curl stack honors the env proxy. If the proxy env is removed, every
auth check "fails" while the profile is actually fine.

## Troubleshooting

| Symptom                                                  | Cause                                        | Fix |
| -------------------------------------------------------- | -------------------------------------------- | --- |
| `Master-token re-mint failed (perform_oauth rejected …)`  | Google revoked the master token (risk flag)  | Section 2 re-auth; if fresh token also dies in minutes → stop, wait ~24h |
| `This browser or app may not be secure` / no oauth_token cookie | Google risk-flagged the CDP Chrome      | Same as above; complete any security challenge inside the window first |
| Auth error but `auth check --test` passes in-container CLI | MCP server started before profile existed (stale client) | `docker compose restart notebooklm-mcp` |
| `406 Not Acceptable: … text/event-stream` on hub route    | Missing Accept header                        | Send `Accept: application/json, text/event-stream` on every hub MCP request |
| `connect ECONNREFUSED :1:9222` / status 500 on /json/version | CDP Chrome died or was launched with wrong flags | Relaunch with the exact flags in section 2 step 1 |
| Shell exits 143 while killing CDP Chrome                  | `pkill -f` matched its own command line      | Kill by `ss`-derived PID instead (never `pkill -f` for this) |
| rsync'd profile dead within ~10 min                       | Local session rotated cookies after the copy | Re-run bootstrap, ship immediately, then respect the single-writer rule |
| Everything authed at bootstrap, dead everywhere minutes later | Google risk-killed a brand-new account    | Wait ~24h, use the account organically in a browser, retry ONCE |
| Hub shows 38 tools but calls return region/redirect errors | Container lost proxy env (recreate)         | Re-add `HTTP(S)_PROXY` to compose env, `up -d` |
| `notebook_create` → "RPC … not found" (rpc_code=5); list works fine | Account never opened NotebookLM in a browser (no first-run init/ToS) — verified fix 2026-11-10 | Open notebooklm.google.com once as the account (through the proxy chrome — HK is region-blocked), accept the welcome flow, then retest create |
| `--backend android` → "needs grpcio"                     | android extra not in the server image         | Not available server-side; web backend is the only one in the container |
| `studio_download` → `VALIDATION: remote file transfer is not configured` | `NOTEBOOKLM_MCP_PUBLIC_URL` unset | Set it (bare https origin) + Caddy TLS front — see "File transfer" section |
| Container exits right after adding `NOTEBOOKLM_MCP_PUBLIC_URL` | Value not a bare https origin (e.g. `http://…`, path suffix) | Use `https://host:port` exactly — no path, no userinfo; validator is fail-closed |
| Set env var in `.env` but tool still says "not configured" | compose `environment:` map enumerates vars explicitly; `env_file`-only entries are ignored | Add the var to the compose `environment:` map too, then `up -d --force-recreate` |
| curl `(60) unable to get local issuer certificate` on signed link | Client machine doesn't trust Caddy's root CA | Arch: cp to `/etc/ca-certificates/trust-source/anchors/` + `sudo trust extract-compat`; Debian: `/usr/local/share/ca-certificates/` + `update-ca-certificates` |
| Signed link 403 "invalid or expired" | Restart since minting, or link TTL passed | Mint a fresh link; key is ephemeral per restart |

## Reference

- Paths: `/opt/notebooklm` + `/opt/caddy` (hades), `~/.notebooklm/profiles/default` (workstation, bootstrap only), `~/.local/share/ca-certificates-extra/caddy-local-root.crt` (workstation, Caddy root CA copy)
- Backups on hades: `docker-compose.yml.orig` (pristine download), `docker-compose.yml.bak-proxy`, `/opt/notebooklm/.env.bak-publicurl`, `docker-compose.yml.bak-publicurl`
- MCPHub admin API: login `POST /api/auth/login` → JWT; auth header `X-Auth-Token`; register
  `POST /api/servers`; check `GET /api/servers/notebooklm`
- MCPHub remote-upstream egress details: `memory/servers/hades/memory.md` (gluetun section)
- Deployment history and rollback notes: `memory/servers/hades/changelog.log` (2026-11-09/10)
- Upstream: https://github.com/teng-lin/notebooklm-py

## Post-Execution Reflection

1. Did the commands succeed? If not, fix the step or troubleshooting row that misled you.
2. Did flags/output drift? Update the tables to match.
3. Workaround needed? Fold it in here so the next run doesn't rediscover it.
