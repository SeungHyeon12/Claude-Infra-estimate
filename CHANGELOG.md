# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed (v0.1.2)
- Slider label "Target RPS" → "Target QPS / RPS / TPS" (the three are interchangeable; UI now matches `load_unit_equivalence`).
- `mockup.html` regenerated from `template/base.html` + REST sample data (was hand-crafted and drifting). New file `mockup-ws.html` for the WebSocket chat demo with fan-out + hard-cap walls.
- Two README screenshots: `docs/demo.png` (REST mode) and `docs/demo-ws.png` (WS chat with hardcap banners + ×N auto-scale visible).
- New "Inputs you can use" section in README cross-checking what the slash command accepts: DAU / MAU / QPS / RPS / TPS / connections / MB/s, domain hints (social/ecom/chat/payments/SaaS), cloud + service pinning, WebSocket-only inputs (msg/sec/conn, fan-out).


### Fixed
- Broken Iconify icons in catalogs (8 names that 404'd on the Iconify CDN). Mapped to working alternatives:
  - `logos:aws-elastic-load-balancing` → `logos:aws-elb`
  - `logos:aws-route-53` → `logos:aws-route53`
  - `logos:aws-x-ray` → `logos:aws-xray`
  - `logos:google-compute-engine` → `logos:google-cloud`
  - `logos:google-cloud-sql` → `logos:postgresql`
  - `logos:google-cloud-firestore` → `logos:firebase`
  - `logos:google-kubernetes-engine` → `logos:kubernetes`
  - `logos:microsoft-sql-server` → `simple-icons:microsoftsqlserver`
- Same fixes applied to `mockup.html`.
- CI now probes every `icon` URL in the catalog and fails on any 404 — prevents future regressions.

### Changed (Plugin packaging)
- Restructured as a **Claude Code plugin** (was a project-local slash command):
  - Added `.claude-plugin/plugin.json` (manifest: name=`capacity-estimator`, version, description, license, repo).
  - Added `.claude-plugin/marketplace.json` so the repo can be added directly with `/plugin marketplace add <git-url>`.
  - Moved `.claude/commands/estimate.md` → `commands/estimate.md` (canonical plugin commands location).
  - Slash command now reads bundled assets via `${CLAUDE_PLUGIN_ROOT}/catalog/...` and `${CLAUDE_PLUGIN_ROOT}/template/base.html`. Output stays in the user's cwd at `./.claude/estimates/`.
- Install: `/plugin marketplace add https://github.com/SeungHyeon12/Claude-Infra-estimate` then `/plugin install estimate@capacity-estimator`. Local dev: `claude --plugin-dir <path>`.

### Added (UX)
- **Bare `/estimate`** now autonomously scans the project (`package.json`, `terraform/`, `k8s/`, `migrations/`, `Dockerfile`, …) to infer workload type, existing instance pins, and DB shape, then proposes a one-line hypothesis and asks one confirm-or-correct question. The previous behavior (immediately ask for load) only fires when the scan turns up nothing.
- **Reports now serve on `http://localhost:11131`** via a tiny `python3 -m http.server` bound to `127.0.0.1:11131` and rooted at `.claude/estimates/`. The slash command writes `<timestamp>.html` + always-current `latest.html`, restarts the server (idempotent), and opens the URL. `file://` is no longer used. Stop the server with `lsof -ti tcp:11131 | xargs kill`.

### Added (WebSocket / chat modeling)
- **Two new sliders** in WS mode: `Msg / sec / conn` (0.1–50, default 1.0) and `Fan-out (1 msg → N delivered)` (1–500, default 1). Both encoded in shareable URL params (`m`, `f`).
- **Downstream amplification gate**: components in the WS flow can now be marked `downstreamMsg: true`. When set, the runtime treats the component as the message-processing layer (cache pub/sub, DB writes) and:
  - amplifies its load by `msgRate × fanout`
  - rates it against `cap.rest` (msg/sec throughput) instead of `cap.ws` (which is connection capacity)
  This separates upstream connection-handling load from downstream message-processing load — the canonical chat fan-out shape.
- **Hard-cap field on catalog services** (`limits.max_connections` + `scales_with_instances` + `remediation`):
  - Added to RDS Postgres r5.large/xlarge, Aurora Serverless v2, EC2 m5.large, ElastiCache Redis r6g.large, Cloud SQL Postgres std-2/std-4, Azure SQL S3/S6.
  - When connection load exceeds the wall, a 🚧 banner appears with the remediation (e.g., "Add RDS Proxy / PgBouncer"). Auto-scale ×N does **not** lift this limit unless `scales_with_instances: true` (the kernel-FD case for compute).
- `meta.initialMsgRate` and `meta.initialFanout` for WS scenarios.
- Slash command updated to mark cache/DB with `downstreamMsg: true` and carry through `limits` from catalog when constructing WS scenarios.

### Added
- **Auto-scale on saturate**: when a fixed-cost component crosses 85% util, capacity is automatically multiplied by ×N (chosen to bring util back to ~60%) and the extra `costMo × (N−1)` is added to the total. Util, latency, break-point, and rank all recompute against the scaled architecture.
- Node `×N` badge: blue/info when auto-scaled (capacity actually multiplied), yellow when the component is quota-bound and can't be auto-scaled (request quota increase), red when raw util > 150%.
- Per-scenario banner now lists every auto-scaled component with the total `+$N/mo` extra cost.
- Detail table capacity column shows `effectiveCap (×N)` after scaling.
- Live rank chip on each scenario header (`#1`/`#2`/`#3`/`#4` by capacity-per-$), updates as you move the slider.
- DAU / MAU / QPS / RPS / TPS as load inputs to `/estimate`.
- `sizing_heuristics` block in `catalog/workloads.json` — DAU→QPS conversion, peak factor (default 3×, event-driven 10×), read/write split by domain, cache-hit defaults, storage (×3 replication + 15% metadata), bandwidth, Little's Law, fan-out tail amplification, N+2 provisioning rule
- `single_server_benchmarks` for catalog sanity-checks (NGINX 10k–100k QPS, Postgres 10k–30k QPS, Redis 100k+ QPS, etc.)
- `load_unit_equivalence` — QPS = RPS = TPS for sizing
- `availability_targets` and `sharding_triggers` reference tables
- Missing-load fallback: ask one focused follow-up; if user replies "default", use workload default and flag with ⚠ banner in `contextDescription`

### Phase 2 (planned)
- M/M/c queuing model for multi-instance compute
- Parallel-path latency (max for fan-out)
- Cold-start modeling for serverless
- Connection-limit modeling for WebSocket

## [0.1.0] — 2026-04-27

### Added
- Initial `/estimate` slash command for Claude Code
- Service catalogs for AWS (~30 services), GCP (~25 services), Azure (~25 services)
- Workload patterns (REST, WebSocket, Streaming) in `catalog/workloads.json`
- Single-file HTML report template (`template/base.html`)
- Static demo (`mockup.html`) — no Claude Code needed
- 4-scenario comparison: AWS REST monolith / GCP REST monolith / Azure REST monolith / AWS Serverless
- Interactive sliders: load / cache hit rate / burst multiplier
- Sortable comparison table
- "Auto-find break point" button
- Side-services badges (S3, SQS, CloudWatch, etc.)
- Share-via-URL state encoding
- JSON Schema validation in CI
