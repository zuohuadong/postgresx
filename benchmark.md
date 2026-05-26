# Benchmark

Generated at: 2026-05-26T23:33:15.924Z

Iterations per case: 2000
Concurrency per case: 16

Services:

- Redis and PostgreSQL run on the same GitHub Actions runner in the benchmark workflow.
- The benchmark workflow runs PostgreSQL 18 with asynchronous I/O enabled via `io_method=worker`.
- The workflow gives both service containers `--cpus 2 --memory 2g`.
- Node.js tests run with `node`; Bun.js tests run with `bun`.
- PostgreSQL columns without `(L1)` use `@postgresx/noredis` with local L1 disabled, so reads hit PostgreSQL. These compare Redis as a service with PostgreSQL as a service.
- PostgreSQL `(L1)` columns enable pgredis in-process memory caching for the hot-read case. That is a valid application-cache mode for Redis replacement, but it measures local process memory plus PostgreSQL backing storage.
- PostgreSQL tables created by pgredis are `UNLOGGED` by default for cache-like workloads, and the workflow sets `synchronous_commit=off` for the benchmark database. Both choices trade crash-time recency guarantees for cache throughput.

## Summary

Ops/sec is higher-is-better. Non-L1 PostgreSQL columns show the service-level backend path; `(L1)` columns show the application hot-read path.

| Operation | Redis | Node PG | Node PG/Redis | Node PG L1 | Node PG L1/Redis | Bun PG | Bun PG/Redis | Bun PG L1 | Bun PG L1/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | 37,359.65 | 6,799.69 | 0.18x | - | - | 15,469.24 | 0.41x | - | - |
| KV write (batch) | 219,677.5 | 41,978.93 | 0.19x | - | - | 64,703.54 | 0.29x | - | - |
| KV read | 43,517.5 | 9,217.15 | 0.21x | - | - | 20,280.37 | 0.47x | - | - |
| KV read (batch) | 300,220.74 | 100,860.87 | 0.34x | - | - | 135,149.65 | 0.45x | - | - |
| KV read (hot cache) | 49,405.11 | 9,663.28 | 0.2x | 1,385,768.02 | 28.05x | 25,933.79 | 0.52x | 684,688.35 | 13.86x |
| Counter increment | 51,123.98 | 9,200.62 | 0.18x | - | - | 13,272.62 | 0.26x | - | - |
| Set add | 58,966.13 | 4,626.04 | 0.08x | - | - | 6,921.68 | 0.12x | - | - |
| Pub/Sub publish | 41,974.85 | 12,624.96 | 0.3x | - | - | 18,375.33 | 0.44x | - | - |

## Details

| Operation | Backend | Iterations | Concurrency | Duration ms | Ops/sec |
| --- | --- | ---: | ---: | ---: | ---: |
| KV write | Node.js + Redis | 2000 | 16 | 53.53 | 37,359.65 |
| KV write (batch) | Node.js + Redis | 2000 | 16 | 9.1 | 219,677.5 |
| KV read | Node.js + Redis | 2000 | 16 | 45.96 | 43,517.5 |
| KV read (batch) | Node.js + Redis | 2000 | 16 | 6.66 | 300,220.74 |
| KV read (hot cache) | Node.js + Redis | 2000 | 16 | 40.48 | 49,405.11 |
| Counter increment | Node.js + Redis | 2000 | 16 | 39.12 | 51,123.98 |
| Set add | Node.js + Redis | 2000 | 16 | 33.92 | 58,966.13 |
| Pub/Sub publish | Node.js + Redis | 2000 | 16 | 47.65 | 41,974.85 |
| KV write | Node.js + PostgreSQL | 2000 | 16 | 294.13 | 6,799.69 |
| KV write (batch) | Node.js + PostgreSQL | 2000 | 16 | 47.64 | 41,978.93 |
| KV read | Node.js + PostgreSQL | 2000 | 16 | 216.99 | 9,217.15 |
| KV read (batch) | Node.js + PostgreSQL | 2000 | 16 | 19.83 | 100,860.87 |
| KV read (hot cache) | Node.js + PostgreSQL | 2000 | 16 | 206.97 | 9,663.28 |
| Counter increment | Node.js + PostgreSQL | 2000 | 16 | 217.38 | 9,200.62 |
| Set add | Node.js + PostgreSQL | 2000 | 16 | 432.34 | 4,626.04 |
| Pub/Sub publish | Node.js + PostgreSQL | 2000 | 16 | 158.42 | 12,624.96 |
| KV read (hot cache) | Node.js + PostgreSQL (L1) | 2000 | 16 | 1.44 | 1,385,768.02 |
| KV write | Bun.js + PostgreSQL | 2000 | 16 | 129.29 | 15,469.24 |
| KV write (batch) | Bun.js + PostgreSQL | 2000 | 16 | 30.91 | 64,703.54 |
| KV read | Bun.js + PostgreSQL | 2000 | 16 | 98.62 | 20,280.37 |
| KV read (batch) | Bun.js + PostgreSQL | 2000 | 16 | 14.8 | 135,149.65 |
| KV read (hot cache) | Bun.js + PostgreSQL | 2000 | 16 | 77.12 | 25,933.79 |
| Counter increment | Bun.js + PostgreSQL | 2000 | 16 | 150.69 | 13,272.62 |
| Set add | Bun.js + PostgreSQL | 2000 | 16 | 288.95 | 6,921.68 |
| Pub/Sub publish | Bun.js + PostgreSQL | 2000 | 16 | 108.84 | 18,375.33 |
| KV read (hot cache) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 2.92 | 684,688.35 |

Notes:

- Redis tests use key prefixes and do not flush the whole database.
- PostgreSQL tests create temporary benchmark tables and drop them at the end.
- Empty `(L1)` cells mean that operation does not use pgredis L1 in the benchmark; L1 is only meaningful for hot cache reads.
- Numbers are intended for regression tracking, not universal database sizing.

References behind benchmark design:

- PostgreSQL `UNLOGGED` tables reduce WAL work for cache-like data, with crash-safety and replication trade-offs: https://www.postgresql.org/docs/current/sql-createtable.html
- `synchronous_commit=off` can improve throughput for noncritical transactions while risking loss of recent acknowledged commits after a crash: https://www.postgresql.org/docs/current/runtime-config-wal.html
- PostgreSQL pipeline mode reduces client/server round trips by sending multiple queries before reading prior results: https://www.postgresql.org/docs/current/libpq-pipeline-mode.html
- PostgreSQL bulk-loading guidance favors batching, transactions, prepared statements, and COPY over many independent INSERTs: https://www.postgresql.org/docs/current/populate.html
