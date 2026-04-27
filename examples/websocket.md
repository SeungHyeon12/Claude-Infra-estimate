# Example — WebSocket chat

## Invocation

```
/estimate WebSocket chat service, 100k concurrent connections, 1 msg/sec per user broadcast to 50-user rooms, AWS vs GCP
```

## What the slash command does

1. Parses: workload=WebSocket, load=100k connections, fan-out=50, msg-rate=1/sec.
2. Constructs scenarios optimized for persistent connections (no CDN passthrough):
   - A: AWS — ALB + ECS Fargate (sticky) × 5 + ElastiCache pub/sub
   - B: GCP — Cloud LB + GKE × 5 + Memorystore pub/sub
   - C: AWS Serverless — API Gateway WebSocket + Lambda + DynamoDB Streams
3. Each scenario sized so 100k connections runs at ~50% of compute capacity.

## What to look at

- **Bottleneck shifts** from compute (REST) to **DB max-connections** in WebSocket. RDS Postgres caps at ~800 client connections — you need PgBouncer or a different DB.
- **Memory** becomes the constraint, not CPU. Each connection ≈ 50KB of socket state on Node, plus app-level state.
- **Fan-out cost**: 1 inbound msg × 50 outbound = 50× the compute cost. Visible as a 50× higher utilization on Redis pub/sub channel.
- **Serverless WebSocket** (API Gateway WS) gets more competitive here — bills per connection-minute, not per request, so 100k idle connections are cheap.

## Caveats

The current MVP doesn't separately model:

- Sticky-session overhead at the LB
- Backpressure when consumers can't keep up
- Heartbeat/ping costs

These are on the Phase 2 roadmap. For now, the estimate is the **steady-state** picture, not the warmup or failure modes.
