---
description: Pre-build architecture sizing & cost estimation. Generates an interactive HTML report comparing 2–4 candidate architectures.
argument-hint: <free-form description of workload + candidate architectures>
---

# /estimate — Pre-build capacity & cost estimator

You are generating an interactive HTML capacity-planning report for the user's
proposed architecture(s), **before any code is written**.

## Paths

This command ships as a **Claude Code plugin**. Two distinct path bases are used:

- **`${CLAUDE_PLUGIN_ROOT}`** — set by Claude Code to this plugin's installed
  location. All plugin-bundled assets (`catalog/*.json`, `template/base.html`,
  `docs/`, `examples/`) live here. **Always read assets via this prefix**, e.g.
  `${CLAUDE_PLUGIN_ROOT}/catalog/aws.json`,
  `${CLAUDE_PLUGIN_ROOT}/template/base.html`.
- **The user's current working directory** (no prefix) — output goes here, in
  `./.claude/estimates/<timestamp>.html`. This keeps generated reports next to
  the project the user is sizing, not buried in the plugin's install dir.
- The user's project files you scan (terraform/k8s/migrations/etc.) are also
  in the user's cwd, with no prefix.

When in doubt: **read** plugin-relative; **write** cwd-relative.

## Inputs

The user passes a free-form description as `$ARGUMENTS`. Examples:

- `REST API for user profile + recent orders, 5k RPS, AWS`
- `Social feed, 2M DAU, AWS vs GCP`
- `Payments, 500 TPS peak, AWS`
- `WebSocket chat, 100k connections, compare AWS vs GCP`
- `Event ingestion 500 MB/s, Kafka vs Kinesis`

### Bare invocation — `/estimate` with no arguments

If `$ARGUMENTS` is empty or whitespace-only, **do not immediately ask the
generic missing-load follow-up**. Instead, autonomously scan the project for
infra/workload signals first, synthesize a one-line hypothesis, and ask **one**
focused confirm-or-correct question.

**Scan list (in order, stop early if signal is strong):**

```
package.json, pnpm-lock.yaml, yarn.lock         → deps reveal workload type:
                                                    socket.io / ws / @nestjs/websockets → WS
                                                    kafkajs / @nestjs/microservices / pulsar → Stream
                                                    express / fastify / nestjs / koa / next → REST (default)
requirements.txt, pyproject.toml, Pipfile       → flask/django/fastapi/aiohttp → REST
                                                    aiokafka / faust → Stream
                                                    websockets / channels → WS
go.mod                                          → gin/echo/fiber/chi → REST; nhooyr/gorilla → WS; segmentio/sarama → Stream
Gemfile, composer.json, build.gradle, pom.xml   → framework signals same as above
Dockerfile, docker-compose*.yml                 → exposed ports, base images
serverless.yml, template.yaml (SAM), samconfig  → existing Lambda layout, runtime, memory
terraform/**/*.tf, *.tf                         → aws_instance / aws_db_instance / aws_lb_*,
                                                    google_compute_instance, azurerm_*,
                                                    aws_lambda_function, aws_dynamodb_table
k8s/**/*.yaml, kustomize/**/*.yaml, helm/       → Deployment containers, requests/limits, HPA min/max
prisma/schema.prisma, drizzle/, sqlx/migrations,
migrations/*.sql, schema.sql, models.py         → DB engine, table shape (users/posts → social;
                                                    products/orders → ecom; messages/rooms → chat)
.env.example, config/*.yml                      → REDIS_URL / DATABASE_URL / KAFKA_BROKERS hints
README.md, CLAUDE.md (top of repo)              → domain hint
```

**Synthesis output** — combine scan findings into a single proposal:

> "Detected: **Node/Express REST API**, Postgres (`migrations/` shows `users`,
> `posts`, `likes` tables → social-feed-ish), Redis. Existing terraform pins
> `t3.medium` × 2 + RDS `db.t3.medium`. **Hypothesis: REST, ~social feed, sized
> against 5k peak RPS, AWS vs GCP.** Confirm load (e.g. `2M DAU`, `5k RPS`,
> `default` to use 5k) or correct anything I got wrong."

**Rules for the bare-invocation flow:**

1. Read at most 5–10 files total — quick BOTE scan, not exhaustive parse.
2. Quote the **exact filenames** that drove the inference so the user can
   correct quickly ("Saw `terraform/db.tf` line 12, `aws_db_instance` of class
   `db.r5.large` — using that as the DB pin").
3. If scan finds **nothing actionable**, fall back to the generic ask from the
   Missing-load fallback section.
4. The user's reply (confirm or correct) becomes the effective `$ARGUMENTS`;
   then continue with the normal Inputs / Procedure flow below.
5. Always reflect the scan-derived defaults in `meta.contextDescription` so the
   generated report says where the assumptions came from.

Parse this description for:

- **Workload type**: REST / WebSocket / Streaming  (default: REST)
- **Load magnitude**, in any of these forms:
  - **DAU / MAU** (e.g. "2M DAU", "10M MAU") → convert to QPS via the heuristics in `catalog/workloads.json` → `sizing_heuristics`
  - **QPS / RPS / TPS** (treat as **interchangeable** — see `load_unit_equivalence`). All map to the catalog's `capacity.rest`.
  - **Concurrent connections** (WebSocket) → maps to `capacity.ws`
  - **MB/s** (Streaming) → maps to `capacity.stream`
- **Cloud(s) to compare**: AWS / GCP / Azure (default: all three + a serverless variant)
- **Domain context**: read profile, payment, social feed, etc. — affects cache strategy, R/W split, peak factor
- **Specific services mentioned**: pin them; otherwise pick reasonable defaults

If anything critical is missing or ambiguous, **ask exactly one focused follow-up question**
before proceeding. Don't ask multiple at once.

### Missing-load fallback

If the user gave **no load number at all** (no DAU, no MAU, no QPS/RPS/TPS, no connections, no MB/s):

1. Ask **exactly one** follow-up: *"How much traffic? Either users (e.g. '500k DAU', '5M MAU') or peak rate (e.g. '3k QPS / RPS / TPS'). Reply 'default' to use the workload's default and tune via the slider."*
2. If they reply **"default"**, **"skip"**, **"don't know"**, or stay silent for one turn:
   - Use `catalog/workloads.json → workloads[<workload>].default_load` (REST=5000 RPS, WS=50000 conn, Stream=200 MB/s).
   - In `meta.contextDescription`, say explicitly: *"⚠ No load given — using default {N} {unit}. Move the slider in the report to your real load to recompute."*
   - Set `meta.initialLoad` to the default; the HTML slider lets them recompute live without re-running the command.
3. If they give MAU only (no stickiness hint), assume `stickiness=0.2` and call it out in `contextDescription`.
4. If they give average QPS (explicitly "average" / "avg" / "sustained"), apply the default `peak_factor=3` to derive peak before sizing — and echo both numbers.

**Never silently fabricate a load number.** Either the user said it, you derived it from DAU+heuristics (with the math echoed), or it's the workload default (with a ⚠ banner).

### Load conversion (apply BEFORE sizing scenarios)

Use `catalog/workloads.json` → `sizing_heuristics` to derive the **peak QPS/RPS/TPS** that scenarios will be sized against. The math is:

```
1. If user gave MAU only:  DAU = MAU × stickiness        (default 0.2; pick by domain)
2. avg_qps  = DAU × actions_per_user_per_day / 86400     (pick actions/user by domain)
3. peak_qps = avg_qps × peak_factor                       (default 3×; 10× for event-driven)
4. If asked separately: read_qps = peak_qps × read_ratio,
                        write_qps = peak_qps × write_ratio  (use read_write_split table)
```

**Always size against `peak_qps`, not average.** Size compute/LB at peak; size DB at `peak_qps × write_ratio` (writes are usually the DB bottleneck because reads can be cached).

If the user already gave QPS/RPS/TPS explicitly, **assume it's peak** unless they say "average" — and call this out in `contextDescription` so they can correct you.

Echo the conversion in `meta.contextDescription`, e.g.:
> "Social feed, 2M DAU × 50 actions/day / 86,400 ≈ 1,160 avg QPS × 3 peak = **3,500 peak RPS**, 99% read / 1% write, cache hit 0.95."

## Procedure

### 0. Precondition — verify plugin assets are present

Before reading any catalog values, **verify the bundled assets are readable**.
The plugin cache may be missing, partially populated, or stale (e.g., interrupted
install, manual `rm -rf`, version mismatch). The command must fail loudly with a
remediation step rather than fabricate sizes from nothing.

Run this check first:

```bash
MISSING=()
for f in catalog/aws.json catalog/gcp.json catalog/azure.json catalog/workloads.json template/base.html; do
  [ -r "${CLAUDE_PLUGIN_ROOT}/$f" ] || MISSING+=("$f")
done
if [ ${#MISSING[@]} -gt 0 ]; then
  echo "❌ Plugin assets missing under ${CLAUDE_PLUGIN_ROOT}:"
  printf '   - %s\n' "${MISSING[@]}"
  echo
  echo "Reinstall: /plugin uninstall estimate@capacity-estimator && /plugin install estimate@capacity-estimator"
  exit 1
fi
```

If any file is missing, **stop the command and surface the message above to the
user** — do not proceed with hardcoded fallback numbers or guess from training
data. The catalog is the source of truth.

### 1. Load the catalog

Read these files **from the plugin install dir** (`${CLAUDE_PLUGIN_ROOT}/`) and
use them as the source of truth for specs, capacity, latency, and cost:

- `${CLAUDE_PLUGIN_ROOT}/catalog/aws.json`
- `${CLAUDE_PLUGIN_ROOT}/catalog/gcp.json`
- `${CLAUDE_PLUGIN_ROOT}/catalog/azure.json`
- `${CLAUDE_PLUGIN_ROOT}/catalog/workloads.json` — also contains:
  - `sizing_heuristics` (DAU→QPS, peak factor, R/W split, cache, Little's Law, fan-out tail, provisioning rule)
  - `single_server_benchmarks` (sanity-check ranges per server type)
  - `load_unit_equivalence` (QPS = RPS = TPS for sizing)
  - `availability_targets`, `sharding_triggers`

Each service has `kind`, `capacity` (per workload), `latency` (p50/p95), and `cost`.

**Sanity-check** any catalog `capacity.rest` against `single_server_benchmarks`. If a value
sits outside the band (e.g., `app_server_qps` outside 1k–10k), prefer the benchmark midpoint
and add a note explaining the adjustment in `contextDescription`.

### 2. Read project context (best-effort)

Look for hints in the user's project to refine the estimate:

```
terraform/**/*.tf, *.tf            → existing infra components
k8s/**/*.yaml, kustomize/**/*.yaml → deployments, HPA limits
docker-compose*.yml                → local dev composition
migrations/*.sql, schema.sql       → indexes, table sizes
prisma/schema.prisma, models.py    → ORM hints
serverless.yml, sam.yaml           → existing Lambda layout
```

If found, prefer the user's actual instance types over generic defaults, and
flag missing indexes that would affect DB capacity assumptions.

If none found, state that and use generic defaults.

### 3. Construct 2–4 candidate scenarios

Default scenario set (when user hasn't pinned):

1. **AWS REST monolith**: CloudFront → ALB → EC2 (3× sized to load) → ElastiCache + RDS
2. **GCP REST monolith**: Cloud CDN → Cloud LB → GCE (sized) → Memorystore + Cloud SQL
3. **Azure REST monolith**: Front Door → App Gateway → App Service → Redis + Azure SQL
4. **AWS Serverless**: CloudFront → API Gateway → Lambda → DynamoDB

For WebSocket: swap compute layer for sticky-session-capable services, drop CDN.
For Streaming: swap to Kinesis/Pub/Sub/Event Hubs and add consumer compute layer.

Size each scenario so steady-state load runs at ~50–60% utilization on the bottleneck
component (so users can see headroom on the slider).

### 4. Compute estimates

For each component:

- `util = effective_load / capacity[workload]`
- Apply queuing penalty if `util > 0.7`: `factor = 1 / (1 - util)`, capped at 12
- Sum component latencies (cache and DB weighted by hit rate)
- p99 ≈ p95 × (1.8 / 2.3 / 4.0) depending on bottleneck util
- Break point = lowest load where any component hits 95% util
- Total cost = sum of monthly cost for each component (use `cost_fn` for usage-billed)

### 5. Generate the HTML report

Read `${CLAUDE_PLUGIN_ROOT}/template/base.html`. Find the line:

```js
window.ESTIMATE_DATA = {{ESTIMATE_DATA_JSON}};
```

Replace `{{ESTIMATE_DATA_JSON}}` with a JSON object of this shape:

```js
{
  "meta": {
    "workload": "rest" | "ws" | "stream",
    "initialLoad": <number>,
    "initialHit": <0..1>,
    "initialBurst": <number>,
    "initialMsgRate": <number>,    // WS only — msg/sec/conn (default 1.0)
    "initialFanout":  <number>,    // WS only — 1 publish → N delivered (default 1)
    "contextDescription": "<echo back the user's request, refined>"
  },
  "scenarios": {
    "A": {
      "label": "AWS — m5.large × 3",
      "cloud": "aws" | "gcp" | "azure" | "other",
      "flow": [
        {
          "id": "<unique key, e.g. 'cdn'>",
          "name": "<display name>",
          "spec": "<one-line spec>",
          "icon": "<iconify name, e.g. logos:aws-ec2>",
          "iconFallback": "<emoji to show if icon fails to load>",
          "cap": { "rest": <num>, "ws": <num>, "stream": <num> },
          "p50": <ms>, "p95": <ms>,
          "costMo": <USD/month>,
          "splitHit": true,    // optional — appears on cache-hit branch only
          "splitMiss": true,   // optional — appears on cache-miss branch only
          "costFn": "load * 0.001",  // optional — JS expression of (load, workload)
          "downstreamMsg": true,     // WS only — rate this against cap.rest, amplify by msgRate × fanout
          "limits": {                 // optional — copy from catalog when present
            "max_connections": 1700,
            "scales_with_instances": false,
            "remediation": "Add RDS Proxy or PgBouncer."
          }
        }
      ],
      "sideServices": [
        { "name": "S3", "icon": "logos:aws-s3", "tooltip": "Static assets" }
      ],
      "recommendations": [
        "<HTML-allowed string with <b> emphasis>"
      ],
      "fixedExtraCostMo": 7
    },
    "B": { ... },
    "C": { ... },
    "D": { ... }
  }
}
```

**Rules for `flow`:**

- Order matters — list components in request-path order.
- Mark cache as `splitHit: true`, primary DB as `splitMiss: true`. Without these
  markers, components are assumed to be on every request.
- `cap` keys must be present for the workload(s) the user cares about. Use a
  large number (or `1e12`) to mean "effectively unlimited" — don't use Infinity
  in JSON; use `1e15` instead, the runtime treats large values as unlimited.
- For **WebSocket / chat** scenarios, mark cache and DB components with
  `downstreamMsg: true`. The runtime then:
  - rates that component against `cap.rest` (msg-per-sec throughput), not `cap.ws`
  - amplifies its load by `msgRate × fanout` (the two new sliders)
  This separates upstream connection-handling load (LB, sticky compute) from the
  downstream message-processing load (cache pub/sub, DB writes).
- Carry through any catalog `limits` field on a service (e.g., `max_connections`)
  by copying it onto the flow component verbatim. The runtime checks WS connection
  count against this hard wall and shows a 🚧 banner with the remediation when
  breached — auto-scale ×N does *not* lift it (unless `scales_with_instances=true`).

**Recommendations:**

- 2–4 actionable suggestions per scenario.
- Each should change ONE thing and quote the resulting metric delta.
  Example: "Scale to 4 × m5.large (+$70/mo) → break point 8.5k RPS"

**Cost functions:**

- Use `costMo` (fixed monthly) for VMs, fixed-tier DBs, LBs.
- Use `costFn` (JS string) for serverless / per-request billing. The string is
  evaluated as `Function('load', 'workload', 'return ' + costFn)`. Example:
  `"workload === 'rest' ? Math.round(load * 86400 * 30 / 1e6 * 1.0) : 0"`

### 6. Write, serve, and open

Save the filled HTML to `./.claude/estimates/{ISO-timestamp}.html` (in the
**user's current working directory**, *not* the plugin install dir — output
should live next to the project the user is sizing). The report **MUST** then
be served via a static HTTP server on `127.0.0.1:11131` and opened in the
user's default browser. **Never** end the command with `file://` or with no
browser open — the user expects to see the rendered page every run.

**Why a server instead of `open <file>`?** Browsers restrict `file://` (clipboard
share-link, future `fetch()`/POST hooks). `localhost` URLs avoid all of that and
are also shareable to teammates on the same machine.

Run these shell steps (idempotent — safe to run on every invocation):

```bash
ESTIMATE_DIR=".claude/estimates"
mkdir -p "$ESTIMATE_DIR"
TS=$(date -u +%Y-%m-%dT%H-%M-%SZ)
OUT="${ESTIMATE_DIR}/${TS}.html"
# (Use the Write tool to write the filled template to $OUT.)
cp "$OUT" "${ESTIMATE_DIR}/latest.html"

# Restart static server on 11131 bound to this project's estimate dir.
# Kill any old server (could belong to a different project) and start fresh.
PID=$(lsof -ti tcp:11131 2>/dev/null || true)
[ -n "$PID" ] && kill "$PID" 2>/dev/null && sleep 0.2 || true
( cd "$ESTIMATE_DIR" && nohup python3 -m http.server 11131 --bind 127.0.0.1 \
    > /tmp/capacity-estimator-server.log 2>&1 & disown ) >/dev/null 2>&1
sleep 0.4

URL="http://localhost:11131/${TS}.html"
open "$URL"
```

If `python3` is not available, fall back to `npx --yes http-server -p 11131 -a 127.0.0.1 "$ESTIMATE_DIR"`.

Print to the console (after the browser opens):

- **URL**: `http://localhost:11131/{TS}.html` (and `…/latest.html` always points at the most recent)
- The on-disk path: `.claude/estimates/{TS}.html`
- A 3-line summary: best capacity-per-$, cheapest, primary bottleneck per scenario
- Caveat: estimates are ±50% — refine catalog values for your account & region
- One-liner to stop the server: `lsof -ti tcp:11131 | xargs kill`

## Style guide

- Be concrete and specific in `recommendations` and bottleneck warnings.
- Don't sandbag — if Lambda at sustained 5k RPS is $19k/mo, say so. The whole
  point is to surface non-obvious tradeoffs.
- Prefer fewer scenarios with deeper analysis over many shallow ones. 4 max.
- Use real region/year pricing from the catalog. If a value seems stale, mention it.
