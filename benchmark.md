# Benchmark

Generated at: 2026-06-20T15:17:43.661Z

Iterations per case: 2000
Concurrency per case: 16

Services:

- Redis and PostgreSQL run on the same GitHub Actions runner in the benchmark workflow.
- The benchmark workflow runs PostgreSQL 18 with asynchronous I/O enabled via `io_method=worker`.
- The workflow gives both service containers `--cpus 2 --memory 2g`.
- Node.js tests run with `node`; Bun.js tests run with `bun`.
- Node.js PostgreSQL uses a connection pool sized to the benchmark concurrency.
- The recommended cache replacement path is L1 in-process memory backed by PostgreSQL L2 storage. L1 rows show that path; L2 rows show the direct PostgreSQL fallback/backend path.
- The 99%, 95%, and 90% L1 rows intentionally mix local hits with PostgreSQL misses to model realistic cache-aside workloads.
- PostgreSQL tables created by pgredis are `UNLOGGED` by default for cache-like workloads, and the workflow sets `synchronous_commit=off` for the benchmark database. Both choices trade crash-time recency guarantees for cache throughput.

## Application Cache Path

Ops/sec is higher-is-better. This table follows the recommended Redis replacement shape: KV reads use L1 when a matching L1 scenario exists; writes and non-cache primitives use the PostgreSQL backend path.

| Operation | Redis | Redis p50 ms | Node PG | Node PG p50 ms | Node PG/Redis | Bun PG | Bun PG p50 ms | Bun PG/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | 23,815.49 | 0.534 | 4,649.73 | 2.75 | 0.2x | 10,192.18 | 1.33 | 0.43x |
| KV write (batch) | 127,176.35 | 1.88 | 29,732.86 | 6.90 | 0.23x | 54,355.21 | 3.90 | 0.43x |
| KV read | 34,803.63 | 0.442 | 6,473.3 | 2.21 | 0.19x | 13,161.31 | 1.01 | 0.38x |
| KV read (batch) | 229,216.01 | 0.882 | 93,665.35 | 2.25 | 0.41x | 88,110.66 | 2.21 | 0.38x |
| KV read (hot cache) L1 | 33,522.29 | 0.441 | 1,203,554.82 | 0.011 | 35.9x | 652,165.55 | 0.021 | 19.45x |
| KV read (99% L1) L1 | 37,569.7 | 0.403 | 453,868.78 | 0.003 | 12.08x | 425,001.39 | 0.008 | 11.31x |
| KV read (95% L1) L1 | 34,761.24 | 0.449 | 170,728.9 | 0.001 | 4.91x | 304,694.12 | 0.001 | 8.77x |
| KV read (90% L1) L1 | 31,356.06 | 0.410 | 130,367.23 | 0.001 | 4.16x | 222,575 | 0.001 | 7.1x |
| Counter increment | 36,726.97 | 0.423 | 7,512 | 1.81 | 0.2x | 12,024.94 | 1.09 | 0.33x |
| Set add | 46,769.91 | 0.307 | 4,180.23 | 3.11 | 0.09x | 6,591.45 | 1.78 | 0.14x |
| Pub/Sub publish | 54,026.42 | 0.292 | 9,522.04 | 1.59 | 0.18x | 15,851.87 | 0.962 | 0.29x |

## L1 Read Cache

These rows isolate pgredis local memory cache behavior. Mixed hit-rate rows include PostgreSQL misses and are closer to real cache-aside usage than the 100% hot-cache row.

| Operation | Redis | Redis p50 ms | Node PG L1 | Node PG L1 p50 ms | Node PG L1/Redis | Bun PG L1 | Bun PG L1 p50 ms | Bun PG L1/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV read (hot cache) | 33,522.29 | 0.441 | 1,203,554.82 | 0.011 | 35.9x | 652,165.55 | 0.021 | 19.45x |
| KV read (99% L1) | 37,569.7 | 0.403 | 453,868.78 | 0.003 | 12.08x | 425,001.39 | 0.008 | 11.31x |
| KV read (95% L1) | 34,761.24 | 0.449 | 170,728.9 | 0.001 | 4.91x | 304,694.12 | 0.001 | 8.77x |
| KV read (90% L1) | 31,356.06 | 0.410 | 130,367.23 | 0.001 | 4.16x | 222,575 | 0.001 | 7.1x |

## L2 Backend Path

These rows disable pgredis L1 and measure direct PostgreSQL access. They are useful for fallback sizing and regression tracking, not as the main cache-hit comparison.

| Operation | Redis | Redis p50 ms | Node PG L2 | Node PG L2 p50 ms | Node PG L2/Redis | Bun PG L2 | Bun PG L2 p50 ms | Bun PG L2/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | 23,815.49 | 0.534 | 4,649.73 | 2.75 | 0.2x | 10,192.18 | 1.33 | 0.43x |
| KV write (batch) | 127,176.35 | 1.88 | 29,732.86 | 6.90 | 0.23x | 54,355.21 | 3.90 | 0.43x |
| KV read | 34,803.63 | 0.442 | 6,473.3 | 2.21 | 0.19x | 13,161.31 | 1.01 | 0.38x |
| KV read (batch) | 229,216.01 | 0.882 | 93,665.35 | 2.25 | 0.41x | 88,110.66 | 2.21 | 0.38x |
| KV read (hot cache) | 33,522.29 | 0.441 | 6,503.04 | 2.23 | 0.19x | 16,526.75 | 0.877 | 0.49x |
| KV read (99% L1) | 37,569.7 | 0.403 | 7,265.86 | 1.97 | 0.19x | 13,829.82 | 1.05 | 0.37x |
| KV read (95% L1) | 34,761.24 | 0.449 | 7,439.16 | 1.94 | 0.21x | 14,740.67 | 0.992 | 0.42x |
| KV read (90% L1) | 31,356.06 | 0.410 | 6,161.01 | 2.33 | 0.2x | 17,854.85 | 0.823 | 0.57x |
| Counter increment | 36,726.97 | 0.423 | 7,512 | 1.81 | 0.2x | 12,024.94 | 1.09 | 0.33x |
| Set add | 46,769.91 | 0.307 | 4,180.23 | 3.11 | 0.09x | 6,591.45 | 1.78 | 0.14x |
| Pub/Sub publish | 54,026.42 | 0.292 | 9,522.04 | 1.59 | 0.18x | 15,851.87 | 0.962 | 0.29x |

## Details

| Operation | Backend | Iterations | Concurrency | Duration ms | Ops/sec | Avg ms | p50 ms | p99 ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | Node.js + Redis | 2000 | 16 | 83.98 | 23,815.49 | 0.663 | 0.534 | 2.15 |
| KV write (batch) | Node.js + Redis | 2000 | 16 | 15.73 | 127,176.35 | 1.86 | 1.88 | 4.62 |
| KV read | Node.js + Redis | 2000 | 16 | 57.47 | 34,803.63 | 0.457 | 0.442 | 1.05 |
| KV read (batch) | Node.js + Redis | 2000 | 16 | 8.73 | 229,216.01 | 1.04 | 0.882 | 2.66 |
| KV read (hot cache) | Node.js + Redis | 2000 | 16 | 59.66 | 33,522.29 | 0.474 | 0.441 | 1.46 |
| KV read (99% L1) | Node.js + Redis | 2000 | 16 | 53.23 | 37,569.7 | 0.422 | 0.403 | 0.972 |
| KV read (95% L1) | Node.js + Redis | 2000 | 16 | 57.54 | 34,761.24 | 0.458 | 0.449 | 0.760 |
| KV read (90% L1) | Node.js + Redis | 2000 | 16 | 63.78 | 31,356.06 | 0.508 | 0.410 | 3.03 |
| Counter increment | Node.js + Redis | 2000 | 16 | 54.46 | 36,726.97 | 0.430 | 0.423 | 0.825 |
| Set add | Node.js + Redis | 2000 | 16 | 42.76 | 46,769.91 | 0.339 | 0.307 | 0.628 |
| Pub/Sub publish | Node.js + Redis | 2000 | 16 | 37.02 | 54,026.42 | 0.294 | 0.292 | 0.428 |
| KV write | Node.js + PostgreSQL | 2000 | 16 | 430.13 | 4,649.73 | 3.44 | 2.75 | 10.03 |
| KV write (batch) | Node.js + PostgreSQL | 2000 | 16 | 67.27 | 29,732.86 | 8.40 | 6.90 | 20.82 |
| KV read | Node.js + PostgreSQL | 2000 | 16 | 308.96 | 6,473.3 | 2.47 | 2.21 | 6.64 |
| KV read (batch) | Node.js + PostgreSQL | 2000 | 16 | 21.35 | 93,665.35 | 2.59 | 2.25 | 6.27 |
| KV read (hot cache) | Node.js + PostgreSQL | 2000 | 16 | 307.55 | 6,503.04 | 2.46 | 2.23 | 5.30 |
| KV read (99% L1) | Node.js + PostgreSQL | 2000 | 16 | 275.26 | 7,265.86 | 2.20 | 1.97 | 5.02 |
| KV read (95% L1) | Node.js + PostgreSQL | 2000 | 16 | 268.85 | 7,439.16 | 2.15 | 1.94 | 5.02 |
| KV read (90% L1) | Node.js + PostgreSQL | 2000 | 16 | 324.62 | 6,161.01 | 2.60 | 2.33 | 6.00 |
| Counter increment | Node.js + PostgreSQL | 2000 | 16 | 266.24 | 7,512 | 2.13 | 1.81 | 4.81 |
| Set add | Node.js + PostgreSQL | 2000 | 16 | 478.44 | 4,180.23 | 3.82 | 3.11 | 15.51 |
| Pub/Sub publish | Node.js + PostgreSQL | 2000 | 16 | 210.04 | 9,522.04 | 1.68 | 1.59 | 3.92 |
| KV read (hot cache) | Node.js + PostgreSQL (L1) | 2000 | 16 | 1.66 | 1,203,554.82 | 0.013 | 0.011 | 0.044 |
| KV read (99% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 4.41 | 453,868.78 | 0.030 | 0.003 | 1.23 |
| KV read (95% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 11.71 | 170,728.9 | 0.092 | 0.001 | 2.76 |
| KV read (90% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 15.34 | 130,367.23 | 0.120 | 0.001 | 3.48 |
| KV write | Bun.js + PostgreSQL | 2000 | 16 | 196.23 | 10,192.18 | 1.56 | 1.33 | 5.58 |
| KV write (batch) | Bun.js + PostgreSQL | 2000 | 16 | 36.8 | 54,355.21 | 4.42 | 3.90 | 12.83 |
| KV read | Bun.js + PostgreSQL | 2000 | 16 | 151.96 | 13,161.31 | 1.21 | 1.01 | 3.77 |
| KV read (batch) | Bun.js + PostgreSQL | 2000 | 16 | 22.7 | 88,110.66 | 2.66 | 2.21 | 8.00 |
| KV read (hot cache) | Bun.js + PostgreSQL | 2000 | 16 | 121.02 | 16,526.75 | 0.964 | 0.877 | 2.49 |
| KV read (99% L1) | Bun.js + PostgreSQL | 2000 | 16 | 144.62 | 13,829.82 | 1.15 | 1.05 | 2.79 |
| KV read (95% L1) | Bun.js + PostgreSQL | 2000 | 16 | 135.68 | 14,740.67 | 1.08 | 0.992 | 2.60 |
| KV read (90% L1) | Bun.js + PostgreSQL | 2000 | 16 | 112.01 | 17,854.85 | 0.894 | 0.823 | 2.24 |
| Counter increment | Bun.js + PostgreSQL | 2000 | 16 | 166.32 | 12,024.94 | 1.33 | 1.09 | 4.22 |
| Set add | Bun.js + PostgreSQL | 2000 | 16 | 303.42 | 6,591.45 | 2.41 | 1.78 | 22.01 |
| Pub/Sub publish | Bun.js + PostgreSQL | 2000 | 16 | 126.17 | 15,851.87 | 1.00 | 0.962 | 2.13 |
| KV read (hot cache) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 3.07 | 652,165.55 | 0.023 | 0.021 | 0.071 |
| KV read (99% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 4.71 | 425,001.39 | 0.036 | 0.008 | 0.304 |
| KV read (95% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 6.56 | 304,694.12 | 0.051 | 0.001 | 1.49 |
| KV read (90% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 8.99 | 222,575 | 0.071 | 0.001 | 1.94 |

Notes:

- Redis tests use key prefixes and do not flush the whole database.
- PostgreSQL tests create temporary benchmark tables and drop them at the end.
- L1 applies only to KV reads. Counter, set, and pub/sub rows are functional replacement paths over PostgreSQL, not local-cache shortcuts.
- Numbers are intended for regression tracking, not universal database sizing.

References behind benchmark design:

- PostgreSQL `UNLOGGED` tables reduce WAL work for cache-like data, with crash-safety and replication trade-offs: https://www.postgresql.org/docs/current/sql-createtable.html
- `synchronous_commit=off` can improve throughput for noncritical transactions while risking loss of recent acknowledged commits after a crash: https://www.postgresql.org/docs/current/runtime-config-wal.html
- PostgreSQL pipeline mode reduces client/server round trips by sending multiple queries before reading prior results: https://www.postgresql.org/docs/current/libpq-pipeline-mode.html
- PostgreSQL bulk-loading guidance favors batching, transactions, prepared statements, and COPY over many independent INSERTs: https://www.postgresql.org/docs/current/populate.html
