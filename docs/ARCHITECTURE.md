# Architecture

## Big picture

```
┌─────────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ user types          │    │ Claude Code      │    │ browser          │
│ /estimate <desc>    │───▶│ runs the slash   │───▶│ opens single     │
│ in Claude Code      │    │ command (.md)    │    │ HTML report      │
└─────────────────────┘    └──────────────────┘    └──────────────────┘
                                    │
                                    │ reads
                                    ▼
                           ┌──────────────────┐
                           │ catalog/*.json   │  ← service specs, prices
                           │ template/base.html│  ← UI shell + math (JS)
                           │ project files    │  ← terraform, k8s, schema
                           └──────────────────┘
```

Three concerns are kept separate:

1. **Catalog** (`catalog/*.json`) — pure data: instance specs, prices, per-workload capacity.
2. **Slash command** (`.claude/commands/estimate.md`) — prompt that turns user description into structured scenarios.
3. **Template** (`template/base.html`) — single-file HTML/CSS/JS that takes scenarios as data and renders interactive UI.

The LLM does **scenario construction**. The browser does **live re-computation** as you move the sliders. Math is the same in both, but the LLM uses it to set initial values; the browser owns interactivity.

## Data flow at runtime

1. User types `/estimate <description>`.
2. Slash command prompt loads catalogs + relevant project files.
3. LLM constructs JSON of shape `{ meta, scenarios: { A, B, C, D } }` (see `template/base.html` for full schema).
4. LLM reads `template/base.html`, replaces `{{ESTIMATE_DATA_JSON}}` with the JSON, writes to `.claude/estimates/<timestamp>.html`.
5. LLM runs `open <path>`. Browser opens the file.
6. JS in the page reads `window.ESTIMATE_DATA`, builds DOM, attaches event listeners, computes initial state.
7. User moves sliders → `render()` re-runs `compute()` for each scenario → DOM updates.

No server. No build step. The HTML file is fully self-contained except for icon SVG fetches from the [Iconify](https://iconify.design) public CDN (with text fallback if offline).

## The `compute()` function

For each component on the request path:

```
load_at_component = effective_load × (1.0 if always-on, hit_rate if cache, miss_rate if DB)
util = load_at_component / capacity[workload]
factor = 1                                if util ≤ 0.7
       = min(1 / (1 - util), 12)          if util > 0.7
p50_effective = baseline_p50 × factor
p95_effective = baseline_p95 × factor
```

Total scenario latency:

```
p50 = Σ (p50_effective_i × weight_i)    where weight depends on splitHit/splitMiss
p95 = Σ (p95_effective_i × weight_i)
p99 = p95 × {1.8 if max_util < 0.5,
             2.3 if max_util < 0.85,
             4.0 otherwise}
```

Break point (the load at which the system starts to fail):

```
break_load = min over all components { capacity[workload] × 0.95 / weight }
```

This is intentionally simple. Phase 2 will add M/M/c queuing for multi-instance compute, parallel-path latency for fan-out, and explicit cold-start modeling.

## Why these specific tradeoffs

**Why a single HTML file?**
- Easy to share (drop in Slack, email, Notion).
- Works offline once loaded (only icons fetch external).
- No build/deploy required.
- Trivially open via `open <file>` without a server.

**Why JS for math (not just LLM-precomputed numbers)?**
- Sliders need to recompute live. LLM-precomputed numbers wouldn't update.
- The math is simple enough to embed.
- Keeps the LLM's job to scenario *construction*, not arithmetic.

**Why ±50% accuracy on purpose?**
- The point is not to predict production. It's to surface order-of-magnitude differences before commitment.
- Higher accuracy would require profiling the user's actual code, which is Phase 3.

## Extending the model

See [EXTENDING.md](EXTENDING.md) for adding clouds, services, and instance types.
See [CATALOG.md](CATALOG.md) for the catalog data format.
