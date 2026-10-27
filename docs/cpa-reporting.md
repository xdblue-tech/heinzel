# CPA Cache Reporting (hades)

Per-model prompt-cache hit-rate reporting for the CPA
gateway (container `cli-proxy-api`) on hades. Status:
**idea + baseline only — nothing installed on hades
yet** (2026-10-25); ad-hoc scans are reproducible
today with the commands in [Reproduction](#reproduction).

## Quick scan (works now, no install)

```bash
# from the workstation; scanner script lives in this
# repo working dir while uncommitted (/tmp/hades/*.py
# locally during the 2026-10-25 session)
scp /tmp/hades/cpa_cache_scan2.py xdblue@hades.home:/tmp/
ssh xdblue@hades.home 'sudo python3 /tmp/cpa_cache_scan2.py'
```

Runtime: ~2-3 min for ~112k log files. Read-only.

## Data source

- `/opt/cpa/logs/*.log` on hades — one file per
  request (~113k files since 2026-09-13, root-owned,
  read with sudo). `main.log` in the same dir is only
  the gin access log; the per-request files are the
  data.
- Relevant filename prefixes: `v1-responses-`,
  `v1-chat-completions-`, `backend-api-codex-responses-`.
- Fields extracted per file (head 8 KB + last 16 KB):
  - `URL:` and `Timestamp:` from the REQUEST INFO
    block (RFC3339 with +08:00 offset)
  - `Session-Id` / `X-Client-Request-Id` header (pi
    and codex clients send it; openai-python chat
    clients do not → those requests can't chain to a
    session)
  - `"model":"…"` from the request body
  - response `usage`: `input_tokens` or
    `prompt_tokens`, and `cached_tokens` (under
    `input_tokens_details` / `prompt_tokens_details`)
- Known gaps: ~2.4k chat-completions bodies are
  zstd-compressed on disk → model unparseable
  (reported as `?`); ~1.4k files have no usage tail; a
  handful of `error-*` files lack usage entirely.

## What the report shows

1. **Per-model hit rate** — requests, request-level
   hit % (requests with `cached_tokens > 0`),
   token-level hit % (`Σ cached / Σ input`), avg input
   tokens.
2. **Hit rate vs gap** — for consecutive requests in
   the same `Session-Id + model` pair, bucket the gap
   since the previous request (0-1m, 1-2m, 2-5m,
   5-10m, 10-15m, 15-30m, 30-60m, 1-2h, 2-4h, 4h+) and
   show hit rate + sample count per bucket. This is
   the closest view these logs give to cache
   retention; the flat per-model hit
   % alone is not.
3. **Blip/re-warm scan** (added 2026-10-25): classify
   misses by cause (cold start, compaction collapse
   input < 0.5× prev, blip = retained-input miss),
   plus median requests-to-recovery (cached ≥ 90% of
   own input) after each non-cold miss.
4. **Top warm hits** — same-session hits at gaps
   ≥ 5 min: gap seconds, cached fraction of input,
   timestamp.

Pair rules: same session AND same model; gap in
`[0, 7d)`; input ≥ 1,024 tokens (OpenAI's minimum
cacheable prefix); `cached_tokens ≥ 0` (usage found).
Miss ≠ eviction: prefix rewrites (compaction), cold
session starts, or credential routing all look like
misses. And a hit ≠ long TTL: intervening requests
from other sessions that share a prefix (pi's static
system prompt + tools block is byte-identical across
pi sessions) can keep a prefix warm without the
session gap reflecting idle time. The gap histogram
measures correlation with idle time, NOT backend
cache TTL — treat window readings as hypotheses
unless a scan also checks for intervening same-prefix
traffic (cross-session prefix matching is not
implemented in these scripts).

## Baseline (2026-10-25 scan)

Request-level hits:

| model | requests | request hit % | token hit % |
|---|---|---|---|
| gpt-6-luna | 14,791 | 97.6 | 97.7 |
| deepseek-flash | 35,606 | 97.1 | 97.4 |
| glm-5.3-flash | 26,097 | 87.6 | 91.1 |
| gpt-6.1-sol | 17,860 | 79.2 | 93.3 |
| gpt-6-sol | 5,121 | 41.4 | 93.1 |
| minimax-m3 | 3,938 | 99.2 | 94.4 |

Hit rate by minutes since previous same-session
request (the warm-window view; sample count per
cell in parentheses — treat cells with n < 20 as
anecdotal):

| gap | gpt-6.1-sol | gpt-6-luna | gpt-6-sol | glm-5.3-flash | deepseek-flash |
|---|---|---|---|---|---|
| 0-1m | 100% (11322) | 100% (13443) | 99% (1922) | 100% (1600) | 99% (1437) |
| 2-5m | 96% (408) | 97% (223) | 92% (59) | 96% (50) | 97% (33) |
| 5-10m | 93% (257) | 89% (38) | 90% (48) | 92% (13) | 100% (7) |
| 15-30m | 91% (147) | 100% (9) | 72% (18) | 17% (6) | 50% (10) |
| 30-60m | 87% (97) | 77% (13) | 69% (13) | 0% (3) | 0% (2) |
| 1-2h | 67% (39) | 64% (11) | 7% (14) | 0% (5) | 0% (1) |

Reading — observations vs hypothesis, kept separate:

- Observed, unambiguous: at 30-60 min gaps the
  OpenAI-family models still hit 69-87% while
  glm/deepseek hit 0-17%; at 1-2h the third-party
  models hit 0% (n's too small to rank OpenAI models
  against each other confidently). gpt-6-sol's low
  overall 41.4% is cold session starts (misses
  concentrate on requests with no same-session
  predecessor), not a short window.
- Hypothesis (not measured): "OpenAI keeps the cache
  warm ~1 h+, third-party models evict after
  ~15-30 min". The pairing controls neither for
  warming by other sessions sharing the static pi
  prefix (plausibly frequent for gpt-6.1-sol, less so
  for the third-party providers) nor for compaction
  rewrites; the true backend window for
gpt-6.1-sol could be shorter AND busier traffic than
  the third-party models', or the providers could
  simply scope caches differently.
- Observed but not attributable: same-session hits at
  4h32m (gpt-6.1-sol, 2026-10-20 06:37 +08, 99% of
  113.5k tokens cached) and 3h20m (gpt-6-sol,
  2026-09-26 04:14, 99%). The 06:37 timing suggests
  OpenAI's off-peak retention extension, but an
  intervening same-prefix request anywhere in the gap
  would produce the same effect — that check has not
  been run.

Anchor: OpenAI documents only the API default ("cleared
after 5-10 minutes… always removed within one hour of
last use", https://openai.com/index/api-prompt-caching/)
; nothing public documents the ChatGPT/Codex-backend
window, and the hades logs do not establish one either
— they bound observed behavior under this host's
traffic mix.

Blip/re-warm baseline (paired chains with Session-Id,
2026-10-25): blip rate 0.3% (gpt-6-luna) to 1.3%
(glm-5.3-flash, deepseek-flash) of paired requests;
median re-warm is 1 request on 4 of 6 models (2 on
luna and 6-sol). Re-warm in one request is the norm
on every provider; clustered post-compaction stutters
are tail events, not provider behavior.

## Install plan (not yet done)

1. Merge the three 2026-10-25 scan scripts
   (`/tmp/hades/cpa_cache_scan{,2,3}.py` locally) into
   one read-only scanner: `/opt/cpa/cache_report.py`
   on hades, root-owned `0644`, flags
   `--since-days N --model REGEX --json`.
2. Run as `sudo python3 /opt/cpa/cache_report.py`.
3. Optional: nightly cron writing
   `/opt/cpa/cache-YYYYMMDD.txt` (decide retention
   before enabling).
4. After install: append one line to
   `memory/servers/hades/memory.md` (CLIProxyAPI
   section) and one changelog entry; drop a
   `logger -t heinzel` headline on hades).

## Reproduction

Already-used commands (read-only):

```bash
scp /tmp/hades/cpa_cache_scan.py xdblue@hades.home:/tmp/
ssh xdblue@hades.home 'sudo python3 /tmp/cpa_cache_scan.py'
# per-record split by session presence:
scp /tmp/hades/cpa_cache_scan3.py …
```

Full detail of the run: `memory/servers/hades/changelog.log`,
entry 2026-10-25. Peer reference:
`memory/servers/hades/memory.md` → CLIProxyAPI section
(log format notes live there too).