# Benchmark

Generated at: 2026-06-20T15:40:04.893Z

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
| KV write | 27,582.8 | 0.494 | 6,166.04 | 2.07 | 0.22x | 12,906.01 | 1.05 | 0.47x |
| KV write (batch) | 149,409.43 | 1.28 | 36,822.61 | 6.07 | 0.25x | 56,494.28 | 4.17 | 0.38x |
| KV read | 37,509.52 | 0.403 | 7,493.27 | 1.91 | 0.2x | 16,568.27 | 0.819 | 0.44x |
| KV read (batch) | 291,848.7 | 0.861 | 74,421.87 | 2.78 | 0.26x | 114,700.64 | 1.73 | 0.39x |
| KV read (hot cache) L1 | 37,331.57 | 0.415 | 1,129,227.05 | 0.013 | 30.25x | 530,483.72 | 0.025 | 14.21x |
| KV read (99% L1) L1 | 40,500.47 | 0.368 | 498,073.45 | 0.004 | 12.3x | 420,185.28 | 0.008 | 10.37x |
| KV read (95% L1) L1 | 38,724.87 | 0.403 | 192,772.03 | 0.001 | 4.98x | 210,802.8 | 0.002 | 5.44x |
| KV read (90% L1) L1 | 35,728.91 | 0.380 | 161,661.13 | 0.001 | 4.52x | 253,511.71 | 0.002 | 7.1x |
| Counter increment | 39,407.81 | 0.389 | 9,125.18 | 1.60 | 0.23x | 14,021.78 | 0.939 | 0.36x |
| Set add | 46,291.67 | 0.324 | 4,107.7 | 2.53 | 0.09x | 6,527.8 | 1.74 | 0.14x |
| Pub/Sub publish | 47,102.06 | 0.330 | 11,899.09 | 1.23 | 0.25x | 15,224.6 | 0.925 | 0.32x |

## L1 Read Cache

These rows isolate pgredis local memory cache behavior. Mixed hit-rate rows include PostgreSQL misses and are closer to real cache-aside usage than the 100% hot-cache row.

| Operation | Redis | Redis p50 ms | Node PG L1 | Node PG L1 p50 ms | Node PG L1/Redis | Bun PG L1 | Bun PG L1 p50 ms | Bun PG L1/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV read (hot cache) | 37,331.57 | 0.415 | 1,129,227.05 | 0.013 | 30.25x | 530,483.72 | 0.025 | 14.21x |
| KV read (99% L1) | 40,500.47 | 0.368 | 498,073.45 | 0.004 | 12.3x | 420,185.28 | 0.008 | 10.37x |
| KV read (95% L1) | 38,724.87 | 0.403 | 192,772.03 | 0.001 | 4.98x | 210,802.8 | 0.002 | 5.44x |
| KV read (90% L1) | 35,728.91 | 0.380 | 161,661.13 | 0.001 | 4.52x | 253,511.71 | 0.002 | 7.1x |

## L2 Backend Path

These rows disable pgredis L1 and measure direct PostgreSQL access. They are useful for fallback sizing and regression tracking, not as the main cache-hit comparison.

| Operation | Redis | Redis p50 ms | Node PG L2 | Node PG L2 p50 ms | Node PG L2/Redis | Bun PG L2 | Bun PG L2 p50 ms | Bun PG L2/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | 27,582.8 | 0.494 | 6,166.04 | 2.07 | 0.22x | 12,906.01 | 1.05 | 0.47x |
| KV write (batch) | 149,409.43 | 1.28 | 36,822.61 | 6.07 | 0.25x | 56,494.28 | 4.17 | 0.38x |
| KV read | 37,509.52 | 0.403 | 7,493.27 | 1.91 | 0.2x | 16,568.27 | 0.819 | 0.44x |
| KV read (batch) | 291,848.7 | 0.861 | 74,421.87 | 2.78 | 0.26x | 114,700.64 | 1.73 | 0.39x |
| KV read (hot cache) | 37,331.57 | 0.415 | 8,681.49 | 1.62 | 0.23x | 18,701.02 | 0.783 | 0.5x |
| KV read (99% L1) | 40,500.47 | 0.368 | 8,703.9 | 1.66 | 0.21x | 18,009.2 | 0.783 | 0.44x |
| KV read (95% L1) | 38,724.87 | 0.403 | 8,828.65 | 1.65 | 0.23x | 17,771.08 | 0.787 | 0.46x |
| KV read (90% L1) | 35,728.91 | 0.380 | 8,948.07 | 1.56 | 0.25x | 19,865.35 | 0.717 | 0.56x |
| Counter increment | 39,407.81 | 0.389 | 9,125.18 | 1.60 | 0.23x | 14,021.78 | 0.939 | 0.36x |
| Set add | 46,291.67 | 0.324 | 4,107.7 | 2.53 | 0.09x | 6,527.8 | 1.74 | 0.14x |
| Pub/Sub publish | 47,102.06 | 0.330 | 11,899.09 | 1.23 | 0.25x | 15,224.6 | 0.925 | 0.32x |

## Details

| Operation | Backend | Iterations | Concurrency | Duration ms | Ops/sec | Avg ms | p50 ms | p99 ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | Node.js + Redis | 2000 | 16 | 72.51 | 27,582.8 | 0.571 | 0.494 | 1.97 |
| KV write (batch) | Node.js + Redis | 2000 | 16 | 13.39 | 149,409.43 | 1.56 | 1.28 | 3.28 |
| KV read | Node.js + Redis | 2000 | 16 | 53.32 | 37,509.52 | 0.424 | 0.403 | 1.10 |
| KV read (batch) | Node.js + Redis | 2000 | 16 | 6.85 | 291,848.7 | 0.813 | 0.861 | 1.06 |
| KV read (hot cache) | Node.js + Redis | 2000 | 16 | 53.57 | 37,331.57 | 0.426 | 0.415 | 0.884 |
| KV read (99% L1) | Node.js + Redis | 2000 | 16 | 49.38 | 40,500.47 | 0.390 | 0.368 | 0.577 |
| KV read (95% L1) | Node.js + Redis | 2000 | 16 | 51.65 | 38,724.87 | 0.411 | 0.403 | 0.613 |
| KV read (90% L1) | Node.js + Redis | 2000 | 16 | 55.98 | 35,728.91 | 0.446 | 0.380 | 2.61 |
| Counter increment | Node.js + Redis | 2000 | 16 | 50.75 | 39,407.81 | 0.401 | 0.389 | 0.676 |
| Set add | Node.js + Redis | 2000 | 16 | 43.2 | 46,291.67 | 0.343 | 0.324 | 0.578 |
| Pub/Sub publish | Node.js + Redis | 2000 | 16 | 42.46 | 47,102.06 | 0.336 | 0.330 | 0.472 |
| KV write | Node.js + PostgreSQL | 2000 | 16 | 324.36 | 6,166.04 | 2.59 | 2.07 | 6.27 |
| KV write (batch) | Node.js + PostgreSQL | 2000 | 16 | 54.31 | 36,822.61 | 6.71 | 6.07 | 15.88 |
| KV read | Node.js + PostgreSQL | 2000 | 16 | 266.91 | 7,493.27 | 2.13 | 1.91 | 5.57 |
| KV read (batch) | Node.js + PostgreSQL | 2000 | 16 | 26.87 | 74,421.87 | 3.35 | 2.78 | 8.95 |
| KV read (hot cache) | Node.js + PostgreSQL | 2000 | 16 | 230.38 | 8,681.49 | 1.84 | 1.62 | 3.86 |
| KV read (99% L1) | Node.js + PostgreSQL | 2000 | 16 | 229.78 | 8,703.9 | 1.83 | 1.66 | 3.96 |
| KV read (95% L1) | Node.js + PostgreSQL | 2000 | 16 | 226.54 | 8,828.65 | 1.81 | 1.65 | 3.87 |
| KV read (90% L1) | Node.js + PostgreSQL | 2000 | 16 | 223.51 | 8,948.07 | 1.78 | 1.56 | 5.06 |
| Counter increment | Node.js + PostgreSQL | 2000 | 16 | 219.17 | 9,125.18 | 1.75 | 1.60 | 4.18 |
| Set add | Node.js + PostgreSQL | 2000 | 16 | 486.89 | 4,107.7 | 3.89 | 2.53 | 36.36 |
| Pub/Sub publish | Node.js + PostgreSQL | 2000 | 16 | 168.08 | 11,899.09 | 1.34 | 1.23 | 3.08 |
| KV read (hot cache) | Node.js + PostgreSQL (L1) | 2000 | 16 | 1.77 | 1,129,227.05 | 0.013 | 0.013 | 0.038 |
| KV read (99% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 4.02 | 498,073.45 | 0.030 | 0.004 | 0.437 |
| KV read (95% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 10.37 | 192,772.03 | 0.080 | 0.001 | 2.31 |
| KV read (90% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 12.37 | 161,661.13 | 0.095 | 0.001 | 2.47 |
| KV write | Bun.js + PostgreSQL | 2000 | 16 | 154.97 | 12,906.01 | 1.23 | 1.05 | 4.48 |
| KV write (batch) | Bun.js + PostgreSQL | 2000 | 16 | 35.4 | 56,494.28 | 4.24 | 4.17 | 9.10 |
| KV read | Bun.js + PostgreSQL | 2000 | 16 | 120.71 | 16,568.27 | 0.961 | 0.819 | 3.03 |
| KV read (batch) | Bun.js + PostgreSQL | 2000 | 16 | 17.44 | 114,700.64 | 2.10 | 1.73 | 7.14 |
| KV read (hot cache) | Bun.js + PostgreSQL | 2000 | 16 | 106.95 | 18,701.02 | 0.852 | 0.783 | 2.12 |
| KV read (99% L1) | Bun.js + PostgreSQL | 2000 | 16 | 111.05 | 18,009.2 | 0.882 | 0.783 | 2.29 |
| KV read (95% L1) | Bun.js + PostgreSQL | 2000 | 16 | 112.54 | 17,771.08 | 0.896 | 0.787 | 2.17 |
| KV read (90% L1) | Bun.js + PostgreSQL | 2000 | 16 | 100.68 | 19,865.35 | 0.803 | 0.717 | 2.25 |
| Counter increment | Bun.js + PostgreSQL | 2000 | 16 | 142.64 | 14,021.78 | 1.13 | 0.939 | 3.49 |
| Set add | Bun.js + PostgreSQL | 2000 | 16 | 306.38 | 6,527.8 | 2.44 | 1.74 | 21.18 |
| Pub/Sub publish | Bun.js + PostgreSQL | 2000 | 16 | 131.37 | 15,224.6 | 1.05 | 0.925 | 3.06 |
| KV read (hot cache) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 3.77 | 530,483.72 | 0.029 | 0.025 | 0.092 |
| KV read (99% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 4.76 | 420,185.28 | 0.036 | 0.008 | 0.209 |
| KV read (95% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 9.49 | 210,802.8 | 0.075 | 0.002 | 2.16 |
| KV read (90% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 7.89 | 253,511.71 | 0.061 | 0.002 | 1.68 |

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
