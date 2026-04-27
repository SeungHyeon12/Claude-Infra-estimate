# Catalog format

Each cloud has a JSON file in `catalog/`:

- `catalog/aws.json`
- `catalog/gcp.json`
- `catalog/azure.json`

Plus `catalog/workloads.json` for workload patterns and `catalog/schema.json` for the JSON Schema.

## Top-level structure

```json
{
  "vendor": "aws" | "gcp" | "azure" | "other",
  "region_default": "us-east-1",
  "currency": "USD",
  "last_updated": "2026-04-27",
  "services": {
    "<service-id>": { ... }
  }
}
```

Service IDs follow the convention `<kind>.<service>.<tier>`, e.g.:

- `compute.ec2.m5.large`
- `database.rds.postgres.r5.xlarge`
- `cache.elasticache.t3.medium`

Use lowercase, dot-separated. The ID is what `.claude/commands/estimate.md` uses to look up entries.

## Service entry

```json
{
  "kind": "compute",
  "display_name": "EC2 m5.large",
  "icon": "logos:aws-ec2",
  "spec": { "vcpu": 2, "ram_gb": 8, "network_gbps": 10 },
  "capacity": {
    "rest": 3000,
    "ws": 30000,
    "stream": 100
  },
  "latency": { "p50": 5, "p95": 15 },
  "cost": { "cost_per_hour_usd": 0.096 },
  "notes": "Optional caveats."
}
```

### `kind`

One of: `compute | database | cache | lb | cdn | queue | storage | stream | auth | monitoring | dns | secrets | function | container`.

This drives placement in the default scenario flow:

```
cdn → lb → (compute | function | container) → cache → database
                                            → queue → stream
```

### `capacity` (per workload)

| key      | unit                             | what it represents                          |
|----------|----------------------------------|---------------------------------------------|
| `rest`   | requests per second              | sustained RPS at p99 < 50ms                  |
| `ws`     | concurrent connections           | conn count at which memory/FDs saturate      |
| `stream` | megabytes per second of ingest   | sustained throughput before consumer lag     |

If a service doesn't apply to a workload, omit that key (or set to 0).

### `latency`

`p50` and `p95` are the **baseline** latency contribution of this hop (in ms), assuming util < 0.7. The runtime applies a queuing penalty above 0.7. **Don't pre-bake queuing into your numbers** — let the model do it.

### `cost`

Pick exactly one of:

- `cost_per_hour_usd` — for hourly-billed VMs/containers/DBs.
- `cost_per_month_usd` — for fixed-tier services.
- `cost_fn` — for usage-billed services (Lambda, DynamoDB on-demand, API Gateway).

`cost_fn` shape:

```json
"cost_fn": {
  "model": "per_request" | "per_gb_sec" | "per_message" | "per_gb_egress" | "per_rcu_wcu",
  "rest":   0.000001,
  "ws":     0.05,
  "stream": 8
}
```

Numbers are dollars per unit of the model. The runtime multiplies by load × time-window. If a service isn't usable for a workload, omit that key.

### `notes`

Free-form. Use for caveats: regional pricing differences, soft limits, version-specific behavior. The LLM will surface notes as "⚠ caveat" banners when relevant.

### `spec`

Free-form object. Common keys:

- `vcpu`, `ram_gb`, `ram_mb`, `network_gbps` — for compute
- `engine` — `postgres` / `mysql` / `redis` / `kv` / `doc` / etc.
- `dtu`, `ru_per_sec` — for service-specific metrics
- `vendor_specific_anything` — fine, just include it

The runtime doesn't read `spec`; it's for human readers and for the LLM to use when explaining tradeoffs.

## Workloads file

`catalog/workloads.json` defines:

- Each workload's load unit, default ranges, common bottlenecks
- Queuing-model rules (M/M/1 factor curve)
- p99 estimation rules
- Burst multiplier guidance
- Cold-start models for serverless runtimes
- **`load_unit_equivalence`** — QPS / RPS / TPS are interchangeable for the `rest` workload
- **`sizing_heuristics`** — DAU→QPS conversion, peak factor, R/W split, cache-hit defaults, storage / bandwidth formulas, Little's Law, fan-out tail amplification, provisioning rule
- **`single_server_benchmarks`** — order-of-magnitude per-server capacity ranges; the slash command sanity-checks catalog values against these
- **`availability_targets`** — nines → downtime budget
- **`sharding_triggers`** — when to recommend partitioning / sharding

If you add a new workload type, both this file and `template/base.html` need updating (UI controls, slider ranges).

## Validation

CI runs:

```bash
npx ajv-cli validate -s catalog/schema.json -d "catalog/aws.json"
npx ajv-cli validate -s catalog/schema.json -d "catalog/gcp.json"
npx ajv-cli validate -s catalog/schema.json -d "catalog/azure.json"
```

Run this locally before opening a PR.
