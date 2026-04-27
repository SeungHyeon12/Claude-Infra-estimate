# Contributing

Thanks for your interest in improving capacity-estimator. The project is most useful when its catalog reflects real-world numbers, so **PRs that add or update catalog entries are especially welcome.**

## Adding a new instance type / service to an existing cloud

1. Open `catalog/aws.json`, `catalog/gcp.json`, or `catalog/azure.json`.
2. Add an entry under `services`. The key should follow the convention `<kind>.<service>.<tier>`, e.g. `compute.ec2.m5.large`.
3. Required fields: `kind`, `display_name`, `cost`. See [`catalog/schema.json`](catalog/schema.json) for the full schema.
4. Latency and capacity values: prefer **measured** numbers from your own benchmarks. If you only have rough rules-of-thumb, say so in `notes`.
5. Pricing: use `cost_per_hour_usd` for VM-style billing, `cost_per_month_usd` for fixed-tier billing, `cost_fn` for per-request billing.
6. Validate locally:
   ```bash
   npx ajv-cli validate -s catalog/schema.json -d "catalog/{aws,gcp,azure}.json"
   ```
7. Open a PR. CI will re-run the validation.

### What counts as a good capacity number?

For `capacity.rest` (RPS), the rule of thumb is "load at which p99 starts to bend":

- **Compute**: ~1500 RPS per modern vCPU for a simple JSON handler. Adjust for runtime (Node ~1.5x of Python).
- **Database**: PK lookup throughput. Complex JOINs reduce by ~3x. Use what your DB shows at p99 < 50ms.
- **Cache**: Redis ops/sec at <2ms p95.
- **LB**: Vendor-published throughput. Most LBs are not the bottleneck.

For `capacity.ws` (concurrent connections):

- **Compute**: Memory-bound. Modern Node holds ~30k connections per vCPU with tuning.
- **DB**: `max_connections` setting (Postgres default 100, Cloud SQL up to 4000).

For `capacity.stream` (MB/s):

- **Compute**: CPU-bound for parsing/transforming. ~50–100 MB/s per vCPU for JSON.
- **Stream services**: Vendor-published throughput per shard/partition.

## Adding a new cloud provider

1. Create `catalog/<vendor>.json` following the same shape.
2. Update `.claude/commands/estimate.md` to list the new vendor in the default scenarios.
3. Update `template/base.html` if the cloud needs a new badge color (search for `cloud-badge.aws`).
4. Add a section to `docs/EXTENDING.md` describing the vendor's pricing peculiarities.
5. Open a PR.

## Improving the estimation model

The math lives in `template/base.html` inside `compute()`. Improvements that have been requested:

- M/M/c queuing model for multi-instance compute (current is M/M/1)
- Parallel-path latency (max instead of sum for fan-out calls)
- Better cold-start integration for serverless (currently buried in catalog notes)
- Multi-region failover modeling

Open an issue first if it's a non-trivial change, so we can discuss the data model implications.

## Reporting bugs

Use GitHub issues. Helpful info:

- The `/estimate` command you ran
- The generated HTML file (zip and attach)
- What was wrong (estimate way off / UI broken / etc.)
- If you have a real-world metric, include it — that's gold for catalog calibration.

## Code style

- HTML/CSS/JS lives in `template/base.html` as a single file. **Keep it single-file.** No build step, no bundler.
- JSON catalogs: use 2-space indent. Keep alphabetical order within `services` (it's not enforced by CI, but it helps reviewing).
- Markdown: prefer ATX headers (`##`), not setext. Keep lines under ~100 chars.
