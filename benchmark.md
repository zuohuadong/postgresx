# Benchmark

Generated at: 2026-05-26T14:45:09.703Z

Iterations per case: 2000
Concurrency per case: 16

Services:

- Redis and PostgreSQL run on the same GitHub Actions runner in the benchmark workflow.
- The benchmark workflow runs PostgreSQL 18 with asynchronous I/O enabled via `io_method=worker`.
- The workflow gives both service containers `--cpus 2 --memory 2g`.
- Node.js tests run with `node`; Bun.js tests run with `bun`.
- PostgreSQL baseline tests use `@postgresx/noredis` with local L1 disabled so reads hit PostgreSQL.
- Rows labeled `(L1)` enable the in-process pgredis L1 cache for the hot-read case. This represents the app-cache path, while non-L1 rows represent PostgreSQL as the remote L2 store.
- PostgreSQL tables created by pgredis are `UNLOGGED` by default for cache-like workloads, and the workflow sets `synchronous_commit=off` for the benchmark database. Both choices trade crash-time recency guarantees for cache throughput.

## Remote L2 Baseline

These rows compare Redis with PostgreSQL when every pgredis read reaches PostgreSQL. This isolates database/driver cost.

| Operation | Node.js + Redis ops/sec | Node.js + PostgreSQL ops/sec | Node/Postgres vs Redis | Bun.js + PostgreSQL ops/sec | Bun/Postgres vs Redis |
| --- | ---: | ---: | ---: | ---: | ---: |
| KV write | 36,284.53 | 7,038.6 | 0.19x | 12,700.35 | 0.35x |
| KV write (batch) | 242,659.08 | 46,090.83 | 0.19x | 63,930.65 | 0.26x |
| KV read | 47,903.81 | 9,388.36 | 0.2x | 17,755.42 | 0.37x |
| KV read (batch) | 342,157.64 | 97,294.93 | 0.28x | 125,335.19 | 0.37x |
| KV read (hot cache) | 51,921.25 | 10,109.37 | 0.19x | 22,216.01 | 0.43x |
| Counter increment | 53,188.12 | 9,368.17 | 0.18x | 11,730.75 | 0.22x |
| Set add | 56,009.27 | 4,440.82 | 0.08x | 6,725.71 | 0.12x |
| Pub/Sub publish | 48,150.82 | 10,214.33 | 0.21x | 16,612.45 | 0.35x |

## Local L1 Hot Cache

These rows enable pgredis L1 for the hot-read workload. This is the fair cache-comparison path when replacing Redis as an application cache.

| Operation | Node.js + Redis ops/sec | Node.js + PostgreSQL L1 ops/sec | Node/Postgres L1 vs Redis | Bun.js + PostgreSQL L1 ops/sec | Bun/Postgres L1 vs Redis |
| --- | ---: | ---: | ---: | ---: | ---: |
| KV read (hot cache) | 51,921.25 | 1,472,678.14 | 28.36x | 555,818.8 | 10.71x |

## Details

| Operation | Backend | Iterations | Concurrency | Duration ms | Ops/sec |
| --- | --- | ---: | ---: | ---: | ---: |
| KV write | Node.js + Redis | 2000 | 16 | 55.12 | 36,284.53 |
| KV write (batch) | Node.js + Redis | 2000 | 16 | 8.24 | 242,659.08 |
| KV read | Node.js + Redis | 2000 | 16 | 41.75 | 47,903.81 |
| KV read (batch) | Node.js + Redis | 2000 | 16 | 5.85 | 342,157.64 |
| KV read (hot cache) | Node.js + Redis | 2000 | 16 | 38.52 | 51,921.25 |
| Counter increment | Node.js + Redis | 2000 | 16 | 37.6 | 53,188.12 |
| Set add | Node.js + Redis | 2000 | 16 | 35.71 | 56,009.27 |
| Pub/Sub publish | Node.js + Redis | 2000 | 16 | 41.54 | 48,150.82 |
| KV write | Node.js + PostgreSQL | 2000 | 16 | 284.15 | 7,038.6 |
| KV write (batch) | Node.js + PostgreSQL | 2000 | 16 | 43.39 | 46,090.83 |
| KV read | Node.js + PostgreSQL | 2000 | 16 | 213.03 | 9,388.36 |
| KV read (batch) | Node.js + PostgreSQL | 2000 | 16 | 20.56 | 97,294.93 |
| KV read (hot cache) | Node.js + PostgreSQL | 2000 | 16 | 197.84 | 10,109.37 |
| Counter increment | Node.js + PostgreSQL | 2000 | 16 | 213.49 | 9,368.17 |
| Set add | Node.js + PostgreSQL | 2000 | 16 | 450.37 | 4,440.82 |
| Pub/Sub publish | Node.js + PostgreSQL | 2000 | 16 | 195.8 | 10,214.33 |
| KV read (hot cache) | Node.js + PostgreSQL (L1) | 2000 | 16 | 1.36 | 1,472,678.14 |
| KV write | Bun.js + PostgreSQL | 2000 | 16 | 157.48 | 12,700.35 |
| KV write (batch) | Bun.js + PostgreSQL | 2000 | 16 | 31.28 | 63,930.65 |
| KV read | Bun.js + PostgreSQL | 2000 | 16 | 112.64 | 17,755.42 |
| KV read (batch) | Bun.js + PostgreSQL | 2000 | 16 | 15.96 | 125,335.19 |
| KV read (hot cache) | Bun.js + PostgreSQL | 2000 | 16 | 90.03 | 22,216.01 |
| Counter increment | Bun.js + PostgreSQL | 2000 | 16 | 170.49 | 11,730.75 |
| Set add | Bun.js + PostgreSQL | 2000 | 16 | 297.37 | 6,725.71 |
| Pub/Sub publish | Bun.js + PostgreSQL | 2000 | 16 | 120.39 | 16,612.45 |
| KV read (hot cache) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 3.6 | 555,818.8 |

Notes:

- Redis tests use key prefixes and do not flush the whole database.
- PostgreSQL tests create temporary benchmark tables and drop them at the end.
- L1 rows are intentionally separated from remote L2 rows because they measure different architectures.
- Numbers are intended for regression tracking, not universal database sizing.

References behind benchmark design:

- PostgreSQL `UNLOGGED` tables reduce WAL work for cache-like data, with crash-safety and replication trade-offs: https://www.postgresql.org/docs/current/sql-createtable.html
- `synchronous_commit=off` can improve throughput for noncritical transactions while risking loss of recent acknowledged commits after a crash: https://www.postgresql.org/docs/current/runtime-config-wal.html
- PostgreSQL pipeline mode reduces client/server round trips by sending multiple queries before reading prior results: https://www.postgresql.org/docs/current/libpq-pipeline-mode.html
- PostgreSQL bulk-loading guidance favors batching, transactions, prepared statements, and COPY over many independent INSERTs: https://www.postgresql.org/docs/current/populate.html
