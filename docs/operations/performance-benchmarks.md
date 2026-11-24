# IRL Engine — Performance Benchmarks

Baseline from the v1.1 harness run of 2026-04-02 on a single-node deployment
(4 vCPU, 8 GB RAM; engine and PostgreSQL on the same host). Each row is the
average of 3 runs. Configuration: `MTA_MODE=mock`, `LAYER2_ENABLED=false`,
`KMS_PROVIDER=none`. Raw run logs are not published. A full run on the current
version is pending.

---

## Throughput

| Concurrency | Throughput | p50 latency | p99 latency |
|---|---|---|---|
| 100 agents | 2,807 req/s | 18 ms | 42 ms |
| 500 agents | 4,097 req/s | 67 ms | 211 ms |
| 1,000 agents | 4,407 req/s | 148 ms | 523 ms |

In these runs the bottleneck was PostgreSQL write throughput (`DB_POOL_MAX_CONNECTIONS=50` at 1,000 agents). The authorize call waits for the database write before it responds.

### Production spot-check (v1.2, 2026-04-08)

A single smoke run on the shared production VPS (2 vCPU, 3.7 GB RAM, other services co-located; `MTA_MODE=none`, `LAYER2_ENABLED=false`; wrk, 2 threads, 5 connections, 30 s). Not a benchmark: different hardware and load.

| Throughput | p50 | p95 | p99 | p99.9 | Errors |
|---|---|---|---|---|---|
| 53 req/s | 75 ms | 93 ms | 100 ms | 106 ms | 0 / 1,580 |

---

## Production Overhead

Live configuration adds latency on top of baseline:

Estimates, not measurements:

| Feature | p50 overhead | p99 overhead |
|---|---|---|
| Live MTA fetch + Ed25519 verify | +2–10 ms | +5–20 ms |
| KMS DEK generation (local) | +5–15 ms | +10–30 ms |
| Heartbeat validation (L2, extra DB read) | +1–3 ms | +2–5 ms |
| All production features combined | ~+10–28 ms | ~+17–55 ms |

---

## Tuning

The primary scaling lever is `DB_POOL_MAX_CONNECTIONS`:

| Concurrency target | Recommended pool size |
|---|---|
| < 50 concurrent agents | 10 (default) |
| 50–200 concurrent agents | 25 |
| 200–500 concurrent agents | 50 |
| 500+ | 100 + read replica via `DB_READONLY_URL` |

Partition maintenance (`pg_partman_bgw`) runs hourly and has negligible throughput impact.

---

## Load Test

Benchmark script: `bench/authorize.lua` (wrk2-compatible).

Run it against your own deployment, not against norve.dev (the public sandbox is rate-limited and shared):

```bash
wrk2 -t4 -c100 -d60s -R 1000 \
  -s bench/authorize.lua \
  https://irl.your-host.example
```

Adjust `-R` (target rate) and `-c` (connections) for your concurrency target.
