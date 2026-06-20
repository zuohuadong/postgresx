# Benchmark

Generated at: 2026-06-20T13:20:09.745Z

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
| KV write | 27,189.84 | 0.510 | 6,020.03 | 2.11 | 0.22x | 12,297.82 | 1.11 | 0.45x |
| KV write (batch) | 142,953.24 | 1.37 | 43,455.49 | 4.70 | 0.3x | 48,934.11 | 4.66 | 0.34x |
| KV read | 38,178.43 | 0.400 | 8,483.12 | 1.72 | 0.22x | 16,371.03 | 0.814 | 0.43x |
| KV read (batch) | 303,184.05 | 0.752 | 92,845.64 | 2.23 | 0.31x | 129,428.33 | 1.55 | 0.43x |
| KV read (hot cache) L1 | 37,269.03 | 0.409 | 1,238,341.79 | 0.011 | 33.23x | 624,005.49 | 0.020 | 16.74x |
| KV read (99% L1) L1 | 42,858.24 | 0.358 | 599,750.86 | 0.003 | 13.99x | 472,071.43 | 0.008 | 11.01x |
| KV read (95% L1) L1 | 41,131.07 | 0.383 | 205,100.84 | 0.001 | 4.99x | 262,551.92 | 0.002 | 6.38x |
| KV read (90% L1) L1 | 36,483.36 | 0.364 | 179,590.78 | 0.001 | 4.92x | 263,415.18 | 0.001 | 7.22x |
| Counter increment | 42,925.25 | 0.357 | 9,227.89 | 1.56 | 0.21x | 13,807.59 | 0.976 | 0.32x |
| Set add | 45,389.33 | 0.328 | 4,048.64 | 2.61 | 0.09x | 6,362.99 | 1.68 | 0.14x |
| Pub/Sub publish | 48,890.48 | 0.322 | 12,367.55 | 1.18 | 0.25x | 17,431.57 | 0.827 | 0.36x |

## L1 Read Cache

These rows isolate pgredis local memory cache behavior. Mixed hit-rate rows include PostgreSQL misses and are closer to real cache-aside usage than the 100% hot-cache row.

| Operation | Redis | Redis p50 ms | Node PG L1 | Node PG L1 p50 ms | Node PG L1/Redis | Bun PG L1 | Bun PG L1 p50 ms | Bun PG L1/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV read (hot cache) | 37,269.03 | 0.409 | 1,238,341.79 | 0.011 | 33.23x | 624,005.49 | 0.020 | 16.74x |
| KV read (99% L1) | 42,858.24 | 0.358 | 599,750.86 | 0.003 | 13.99x | 472,071.43 | 0.008 | 11.01x |
| KV read (95% L1) | 41,131.07 | 0.383 | 205,100.84 | 0.001 | 4.99x | 262,551.92 | 0.002 | 6.38x |
| KV read (90% L1) | 36,483.36 | 0.364 | 179,590.78 | 0.001 | 4.92x | 263,415.18 | 0.001 | 7.22x |

## L2 Backend Path

These rows disable pgredis L1 and measure direct PostgreSQL access. They are useful for fallback sizing and regression tracking, not as the main cache-hit comparison.

| Operation | Redis | Redis p50 ms | Node PG L2 | Node PG L2 p50 ms | Node PG L2/Redis | Bun PG L2 | Bun PG L2 p50 ms | Bun PG L2/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | 27,189.84 | 0.510 | 6,020.03 | 2.11 | 0.22x | 12,297.82 | 1.11 | 0.45x |
| KV write (batch) | 142,953.24 | 1.37 | 43,455.49 | 4.70 | 0.3x | 48,934.11 | 4.66 | 0.34x |
| KV read | 38,178.43 | 0.400 | 8,483.12 | 1.72 | 0.22x | 16,371.03 | 0.814 | 0.43x |
| KV read (batch) | 303,184.05 | 0.752 | 92,845.64 | 2.23 | 0.31x | 129,428.33 | 1.55 | 0.43x |
| KV read (hot cache) | 37,269.03 | 0.409 | 9,270.35 | 1.61 | 0.25x | 20,463.71 | 0.732 | 0.55x |
| KV read (99% L1) | 42,858.24 | 0.358 | 9,414.17 | 1.56 | 0.22x | 21,416.69 | 0.663 | 0.5x |
| KV read (95% L1) | 41,131.07 | 0.383 | 9,355.75 | 1.59 | 0.23x | 20,628.27 | 0.706 | 0.5x |
| KV read (90% L1) | 36,483.36 | 0.364 | 9,425.56 | 1.51 | 0.26x | 19,016.62 | 0.765 | 0.52x |
| Counter increment | 42,925.25 | 0.357 | 9,227.89 | 1.56 | 0.21x | 13,807.59 | 0.976 | 0.32x |
| Set add | 45,389.33 | 0.328 | 4,048.64 | 2.61 | 0.09x | 6,362.99 | 1.68 | 0.14x |
| Pub/Sub publish | 48,890.48 | 0.322 | 12,367.55 | 1.18 | 0.25x | 17,431.57 | 0.827 | 0.36x |

## Details

| Operation | Backend | Iterations | Concurrency | Duration ms | Ops/sec | Avg ms | p50 ms | p99 ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | Node.js + Redis | 2000 | 16 | 73.56 | 27,189.84 | 0.580 | 0.510 | 1.82 |
| KV write (batch) | Node.js + Redis | 2000 | 16 | 13.99 | 142,953.24 | 1.63 | 1.37 | 3.95 |
| KV read | Node.js + Redis | 2000 | 16 | 52.39 | 38,178.43 | 0.414 | 0.400 | 0.706 |
| KV read (batch) | Node.js + Redis | 2000 | 16 | 6.6 | 303,184.05 | 0.771 | 0.752 | 1.13 |
| KV read (hot cache) | Node.js + Redis | 2000 | 16 | 53.66 | 37,269.03 | 0.426 | 0.409 | 1.03 |
| KV read (99% L1) | Node.js + Redis | 2000 | 16 | 46.67 | 42,858.24 | 0.371 | 0.358 | 0.643 |
| KV read (95% L1) | Node.js + Redis | 2000 | 16 | 48.63 | 41,131.07 | 0.387 | 0.383 | 0.666 |
| KV read (90% L1) | Node.js + Redis | 2000 | 16 | 54.82 | 36,483.36 | 0.436 | 0.364 | 2.83 |
| Counter increment | Node.js + Redis | 2000 | 16 | 46.59 | 42,925.25 | 0.369 | 0.357 | 0.675 |
| Set add | Node.js + Redis | 2000 | 16 | 44.06 | 45,389.33 | 0.349 | 0.328 | 0.501 |
| Pub/Sub publish | Node.js + Redis | 2000 | 16 | 40.91 | 48,890.48 | 0.325 | 0.322 | 0.464 |
| KV write | Node.js + PostgreSQL | 2000 | 16 | 332.22 | 6,020.03 | 2.65 | 2.11 | 7.28 |
| KV write (batch) | Node.js + PostgreSQL | 2000 | 16 | 46.02 | 43,455.49 | 5.53 | 4.70 | 15.86 |
| KV read | Node.js + PostgreSQL | 2000 | 16 | 235.76 | 8,483.12 | 1.88 | 1.72 | 4.67 |
| KV read (batch) | Node.js + PostgreSQL | 2000 | 16 | 21.54 | 92,845.64 | 2.63 | 2.23 | 7.01 |
| KV read (hot cache) | Node.js + PostgreSQL | 2000 | 16 | 215.74 | 9,270.35 | 1.72 | 1.61 | 4.13 |
| KV read (99% L1) | Node.js + PostgreSQL | 2000 | 16 | 212.45 | 9,414.17 | 1.70 | 1.56 | 3.76 |
| KV read (95% L1) | Node.js + PostgreSQL | 2000 | 16 | 213.77 | 9,355.75 | 1.71 | 1.59 | 3.74 |
| KV read (90% L1) | Node.js + PostgreSQL | 2000 | 16 | 212.19 | 9,425.56 | 1.69 | 1.51 | 4.23 |
| Counter increment | Node.js + PostgreSQL | 2000 | 16 | 216.73 | 9,227.89 | 1.73 | 1.56 | 5.07 |
| Set add | Node.js + PostgreSQL | 2000 | 16 | 493.99 | 4,048.64 | 3.94 | 2.61 | 38.64 |
| Pub/Sub publish | Node.js + PostgreSQL | 2000 | 16 | 161.71 | 12,367.55 | 1.29 | 1.18 | 2.63 |
| KV read (hot cache) | Node.js + PostgreSQL (L1) | 2000 | 16 | 1.62 | 1,238,341.79 | 0.012 | 0.011 | 0.039 |
| KV read (99% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 3.33 | 599,750.86 | 0.025 | 0.003 | 0.450 |
| KV read (95% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 9.75 | 205,100.84 | 0.076 | 0.001 | 2.30 |
| KV read (90% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 11.14 | 179,590.78 | 0.088 | 0.001 | 2.32 |
| KV write | Bun.js + PostgreSQL | 2000 | 16 | 162.63 | 12,297.82 | 1.29 | 1.11 | 3.98 |
| KV write (batch) | Bun.js + PostgreSQL | 2000 | 16 | 40.87 | 48,934.11 | 4.90 | 4.66 | 10.51 |
| KV read | Bun.js + PostgreSQL | 2000 | 16 | 122.17 | 16,371.03 | 0.971 | 0.814 | 4.07 |
| KV read (batch) | Bun.js + PostgreSQL | 2000 | 16 | 15.45 | 129,428.33 | 1.88 | 1.55 | 4.71 |
| KV read (hot cache) | Bun.js + PostgreSQL | 2000 | 16 | 97.73 | 20,463.71 | 0.778 | 0.732 | 1.83 |
| KV read (99% L1) | Bun.js + PostgreSQL | 2000 | 16 | 93.39 | 21,416.69 | 0.742 | 0.663 | 1.97 |
| KV read (95% L1) | Bun.js + PostgreSQL | 2000 | 16 | 96.95 | 20,628.27 | 0.774 | 0.706 | 2.10 |
| KV read (90% L1) | Bun.js + PostgreSQL | 2000 | 16 | 105.17 | 19,016.62 | 0.840 | 0.765 | 2.17 |
| Counter increment | Bun.js + PostgreSQL | 2000 | 16 | 144.85 | 13,807.59 | 1.15 | 0.976 | 3.48 |
| Set add | Bun.js + PostgreSQL | 2000 | 16 | 314.32 | 6,362.99 | 2.50 | 1.68 | 28.47 |
| Pub/Sub publish | Bun.js + PostgreSQL | 2000 | 16 | 114.73 | 17,431.57 | 0.914 | 0.827 | 2.09 |
| KV read (hot cache) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 3.21 | 624,005.49 | 0.025 | 0.020 | 0.075 |
| KV read (99% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 4.24 | 472,071.43 | 0.032 | 0.008 | 0.290 |
| KV read (95% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 7.62 | 262,551.92 | 0.056 | 0.002 | 1.58 |
| KV read (90% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 7.59 | 263,415.18 | 0.058 | 0.001 | 1.68 |

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
