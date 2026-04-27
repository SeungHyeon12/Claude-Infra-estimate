# Example — Streaming ingestion

## Invocation

```
/estimate Event ingestion 500 MB/s, 5KB avg record, 1 consumer group writing to OLAP, compare Kinesis vs Kafka vs Pub/Sub
```

## What the slash command does

1. Parses: workload=Streaming, throughput=500 MB/s, record-size=5KB, consumers=1.
2. Computes implied partition count: `500 MB/s ÷ 1 MB/s/shard = 500 shards (Kinesis)`, `~50 partitions for Kafka m5.large brokers`.
3. Constructs 3 scenarios:
   - A: AWS — Kinesis Data Streams (500 shards) + Lambda consumers + DynamoDB
   - B: AWS — MSK Kafka (m5.large × 6 brokers) + EKS consumers + DynamoDB
   - C: GCP — Pub/Sub + Dataflow + BigQuery

## What to look at

- **Cost spread is wild**: Kinesis at 500 shards is ~$5,500/mo before PUT charges. Kafka MSK is ~$900/mo for 6 brokers + storage. Pub/Sub is usage-billed at $40/TB published.
- **Consumer lag** is the real metric — visible in the report as "stream consumer p99". If consumers can't process at 500 MB/s, lag grows unboundedly.
- **Partition strategy** affects break point: Kinesis caps at 1 MB/s/shard hard. Kafka can push 50–100 MB/s/broker if tuned.
- **Down-the-stream cost**: writing 500 MB/s to DynamoDB is enormous. Consider whether you need a hot store at all, or if batched writes to S3 + Athena query is enough.

## Recommendations the report typically surfaces

- "Kinesis on-demand mode" instead of provisioned shards if traffic is bursty (auto-scales but slower)
- "Compress records 5KB → ~1KB" before publish → 5× headroom on shards
- "Use Firehose to S3" instead of Lambda consumers if downstream is OLAP
