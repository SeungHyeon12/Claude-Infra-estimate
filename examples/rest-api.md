# Example — REST API

## Invocation

```
/estimate REST API for /users/:id returning profile + last 5 orders, expected 5k RPS, AWS-first but open to GCP
```

## What the slash command does

1. Parses: workload=REST, load=5k RPS, primary=AWS, also evaluate GCP.
2. Looks for project hints:
   - `terraform/main.tf` → finds `aws_db_instance` resource → uses RDS Postgres.
   - `migrations/0001_init.sql` → checks for `(user_id, created_at DESC)` composite index on orders. Flags if missing.
3. Constructs 4 scenarios:
   - A: AWS REST (CloudFront → ALB → EC2 m5.large × 3 → ElastiCache + RDS)
   - B: GCP REST (Cloud CDN → Cloud LB → GCE n2-standard-2 × 3 → Memorystore + Cloud SQL)
   - C: AWS Serverless (CloudFront → API Gateway → Lambda → DynamoDB)
   - D: AWS scale-up (CloudFront → ALB → EC2 m5.xlarge × 2 → ElastiCache + RDS)
4. Renders `.claude/estimates/2026-04-27T10-30-00.html` and opens it.

## Expected output highlights

- **Best capacity per $**: usually B (GCP scale-up) due to slightly cheaper compute
- **Cheapest at low traffic**: C (Serverless) — ~$80/mo if RPS < 200
- **Cheapest at high traffic**: A or D — $620/mo at 5k RPS sustained
- **Highest p99**: C (cold starts) — surprising at first, but visible on the slider when RPS drops

## What to look at in the report

1. Move the **RPS slider** from 500 to 50,000 — see at which point the EC2 bar turns red.
2. Drop **cache hit rate** from 80% to 0% — see the RDS bar turn red.
3. Switch to a **GCP-only or Azure-only view** (visually) by scrolling.
4. Click **"Auto-find break point"** to jump to the saturation RPS.
5. **Sort the table by p99** to see which scenario degrades worst under load.
