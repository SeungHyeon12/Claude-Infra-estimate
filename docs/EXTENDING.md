# Extending capacity-estimator

## Adding an instance type

Most common contribution. Drop a new entry in `catalog/<vendor>.json` under `services`.

Example — adding `c5.4xlarge`:

```json
"compute.ec2.c5.4xlarge": {
  "kind": "compute",
  "display_name": "EC2 c5.4xlarge (compute-opt)",
  "icon": "logos:aws-ec2",
  "spec": { "vcpu": 16, "ram_gb": 32, "network_gbps": 10 },
  "capacity": { "rest": 32000, "ws": 200000, "stream": 1200 },
  "latency": { "p50": 4, "p95": 12 },
  "cost": { "cost_per_hour_usd": 0.68 }
}
```

Capacity rule of thumb:

- `rest`: ~2,000 RPS per modern vCPU for simple JSON. Multiply for compute-optimized, divide ~2 for burstable.
- `ws`: ~12,500 concurrent connections per vCPU (memory-bound).
- `stream`: ~75 MB/s per vCPU for JSON parsing, more for binary.

Tune from there if you have measured values. **Real measurements > heuristics.** Add a `notes` field stating the source.

## Adding a service

For a service kind we already support (`database`, `cache`, etc.):

1. Pick the right `kind`.
2. Pick a service ID using the convention `<kind>.<vendor-name>.<tier>`.
3. Fill in spec/capacity/latency/cost.
4. If the service has a quirky pricing model, use `cost_fn`. Document the model in `notes`.

Example — adding ElastiCache Redis Cluster mode:

```json
"cache.elasticache.cluster.r6g.large": {
  "kind": "cache",
  "display_name": "ElastiCache Redis Cluster r6g.large × 3 shards",
  "icon": "logos:aws-elasticache",
  "spec": { "vcpu": 2, "ram_gb": 13.07, "engine": "redis-cluster", "shards": 3 },
  "capacity": { "rest": 300000, "ws": 180000, "stream": 6000 },
  "latency": { "p50": 0.7, "p95": 2.0 },
  "cost": { "cost_per_hour_usd": 0.603 },
  "notes": "Cluster mode adds ~0.5ms p95 vs single-shard. 3-shard min for cluster."
}
```

## Adding a new cloud provider

For a new vendor (Oracle Cloud, Alibaba, Cloudflare, on-prem):

1. Create `catalog/<vendor>.json` with the standard structure (see `catalog/schema.json`).
2. Add at least one service per kind your users might use: `compute`, `database`, `cache`, `lb`, `cdn`, optional `queue`/`stream`.
3. In `template/base.html`, add a CSS rule for the cloud badge color:

```css
.cloud-badge.cloudflare { background: #f38020; color: #fff; }
.cmp-table .pill.cloudflare { background: #f38020; color: #fff; }
```

4. Update `.claude/commands/estimate.md`:
   - Add the vendor to the list of catalog files to load
   - Add a default scenario template using its services

5. Add a section to this file describing the vendor's pricing peculiarities.

## Adding a new workload type

A new workload (e.g., `batch`, `transactional-write-heavy`, `ml-inference`) is a bigger change:

1. Add the workload to `catalog/workloads.json` with default ranges and bottleneck guidance.
2. Add a `cap.<workload>` value to every relevant service in `catalog/*.json`.
3. In `template/base.html`:
   - Add a button to the workload selector
   - Add a slider input with the right range
   - Update `WORKLOADS` const in JS
   - Update `body[data-workload="..."] .control[data-workload-only="..."]` CSS rules
4. Update `.claude/commands/estimate.md` to handle the new workload's parsing and default scenarios.

## Tuning the estimation math

Math lives in `template/base.html`'s `compute()` function. Improvements to consider:

### M/M/c queuing (multi-instance compute)

Currently the queuing penalty assumes a single queue (M/M/1). For N-instance compute, the M/M/c formula is more accurate:

```
P(wait) = (c·ρ)^c / (c! · (1 - ρ))   where ρ = util
E(wait) = P(wait) · 1 / (c · μ · (1 - ρ))
```

Implementation hint: store `instances` count in the component, plug into Erlang-C formula.

### Parallel-path latency

Currently latencies are summed (assumes serial). For fan-out (e.g., calling 3 microservices in parallel), use `max()` instead. Mark such components with `parallel: true` in the data and adjust `compute()`.

### Cold-start integration

`catalog/workloads.json` defines cold-start latencies per runtime, but `compute()` doesn't currently use them. To integrate:

```js
if (c.kind === 'function' && load < c.cap.rest * 0.01) {
  const coldP95 = WORKLOADS_DATA.cold_start_models[c.runtime]?.p95_ms || 0;
  c.p95Eff += coldP95 * 0.05;  // 5% of requests are cold
}
```

### Connection-limit modeling (WebSocket)

Currently WebSocket capacity is just `cap.ws`. To model the full picture:

- `kernel_fd_limit` per instance
- `lb_sticky_session_overhead` (extra LB cost per persistent connection)
- `db_max_connections` (Postgres `max_connections`, etc.)

Add these as optional `limits` fields on services and check them as additional bottleneck candidates.

## Testing changes

For now, the test is "does the mockup still work and look right?"

```bash
open mockup.html
```

A proper test suite is on the roadmap — should be straightforward Jest-style tests against `compute()` with known inputs/outputs.

## Releasing

The project follows [SemVer](https://semver.org).

- **Patch** (0.1.x): catalog updates, doc fixes, small UI fixes
- **Minor** (0.x.0): new workload types, new estimation rules, new template features
- **Major** (x.0.0): breaking changes to the catalog schema or scenario JSON shape

Bump version in `CHANGELOG.md` and tag the commit:

```bash
git tag v0.1.0
git push --tags
```

GitHub Releases happen automatically from tags via a (planned) workflow.
