# Benchmark

Generated at: 2026-06-20T15:20:11.595Z

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
| KV write | 29,484.67 | 0.464 | 5,862.61 | 2.09 | 0.2x | 13,019.03 | 1.05 | 0.44x |
| KV write (batch) | 154,622.78 | 1.37 | 40,493.15 | 5.59 | 0.26x | 58,776.75 | 4.11 | 0.38x |
| KV read | 39,250.41 | 0.395 | 8,157.99 | 1.76 | 0.21x | 18,026.22 | 0.771 | 0.46x |
| KV read (batch) | 296,306.79 | 0.685 | 92,181.88 | 2.35 | 0.31x | 124,657.39 | 1.67 | 0.42x |
| KV read (hot cache) L1 | 38,123.88 | 0.395 | 792,029.65 | 0.017 | 20.78x | 548,322.12 | 0.025 | 14.38x |
| KV read (99% L1) L1 | 43,273.5 | 0.349 | 460,728.85 | 0.004 | 10.65x | 447,818.61 | 0.008 | 10.35x |
| KV read (95% L1) L1 | 41,062.26 | 0.391 | 208,975.5 | 0.001 | 5.09x | 256,985.97 | 0.001 | 6.26x |
| KV read (90% L1) L1 | 37,496.31 | 0.361 | 174,288.31 | 0.001 | 4.65x | 242,428.44 | 0.001 | 6.47x |
| Counter increment | 46,310.05 | 0.326 | 9,462.41 | 1.53 | 0.2x | 11,384.53 | 1.16 | 0.25x |
| Set add | 49,032.82 | 0.304 | 4,067.02 | 2.48 | 0.08x | 6,422.89 | 1.76 | 0.13x |
| Pub/Sub publish | 53,462.63 | 0.292 | 12,290.79 | 1.13 | 0.23x | 16,834.9 | 0.887 | 0.31x |

## L1 Read Cache

These rows isolate pgredis local memory cache behavior. Mixed hit-rate rows include PostgreSQL misses and are closer to real cache-aside usage than the 100% hot-cache row.

| Operation | Redis | Redis p50 ms | Node PG L1 | Node PG L1 p50 ms | Node PG L1/Redis | Bun PG L1 | Bun PG L1 p50 ms | Bun PG L1/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV read (hot cache) | 38,123.88 | 0.395 | 792,029.65 | 0.017 | 20.78x | 548,322.12 | 0.025 | 14.38x |
| KV read (99% L1) | 43,273.5 | 0.349 | 460,728.85 | 0.004 | 10.65x | 447,818.61 | 0.008 | 10.35x |
| KV read (95% L1) | 41,062.26 | 0.391 | 208,975.5 | 0.001 | 5.09x | 256,985.97 | 0.001 | 6.26x |
| KV read (90% L1) | 37,496.31 | 0.361 | 174,288.31 | 0.001 | 4.65x | 242,428.44 | 0.001 | 6.47x |

## L2 Backend Path

These rows disable pgredis L1 and measure direct PostgreSQL access. They are useful for fallback sizing and regression tracking, not as the main cache-hit comparison.

| Operation | Redis | Redis p50 ms | Node PG L2 | Node PG L2 p50 ms | Node PG L2/Redis | Bun PG L2 | Bun PG L2 p50 ms | Bun PG L2/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | 29,484.67 | 0.464 | 5,862.61 | 2.09 | 0.2x | 13,019.03 | 1.05 | 0.44x |
| KV write (batch) | 154,622.78 | 1.37 | 40,493.15 | 5.59 | 0.26x | 58,776.75 | 4.11 | 0.38x |
| KV read | 39,250.41 | 0.395 | 8,157.99 | 1.76 | 0.21x | 18,026.22 | 0.771 | 0.46x |
| KV read (batch) | 296,306.79 | 0.685 | 92,181.88 | 2.35 | 0.31x | 124,657.39 | 1.67 | 0.42x |
| KV read (hot cache) | 38,123.88 | 0.395 | 8,999.57 | 1.61 | 0.24x | 20,677.9 | 0.721 | 0.54x |
| KV read (99% L1) | 43,273.5 | 0.349 | 9,042.84 | 1.60 | 0.21x | 18,845.99 | 0.778 | 0.44x |
| KV read (95% L1) | 41,062.26 | 0.391 | 9,545.94 | 1.55 | 0.23x | 19,334.8 | 0.740 | 0.47x |
| KV read (90% L1) | 37,496.31 | 0.361 | 8,725.96 | 1.60 | 0.23x | 19,791.92 | 0.740 | 0.53x |
| Counter increment | 46,310.05 | 0.326 | 9,462.41 | 1.53 | 0.2x | 11,384.53 | 1.16 | 0.25x |
| Set add | 49,032.82 | 0.304 | 4,067.02 | 2.48 | 0.08x | 6,422.89 | 1.76 | 0.13x |
| Pub/Sub publish | 53,462.63 | 0.292 | 12,290.79 | 1.13 | 0.23x | 16,834.9 | 0.887 | 0.31x |

## Details

| Operation | Backend | Iterations | Concurrency | Duration ms | Ops/sec | Avg ms | p50 ms | p99 ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | Node.js + Redis | 2000 | 16 | 67.83 | 29,484.67 | 0.535 | 0.464 | 2.19 |
| KV write (batch) | Node.js + Redis | 2000 | 16 | 12.93 | 154,622.78 | 1.50 | 1.37 | 2.86 |
| KV read | Node.js + Redis | 2000 | 16 | 50.95 | 39,250.41 | 0.405 | 0.395 | 0.647 |
| KV read (batch) | Node.js + Redis | 2000 | 16 | 6.75 | 296,306.79 | 0.812 | 0.685 | 2.22 |
| KV read (hot cache) | Node.js + Redis | 2000 | 16 | 52.46 | 38,123.88 | 0.417 | 0.395 | 1.32 |
| KV read (99% L1) | Node.js + Redis | 2000 | 16 | 46.22 | 43,273.5 | 0.367 | 0.349 | 0.531 |
| KV read (95% L1) | Node.js + Redis | 2000 | 16 | 48.71 | 41,062.26 | 0.388 | 0.391 | 0.551 |
| KV read (90% L1) | Node.js + Redis | 2000 | 16 | 53.34 | 37,496.31 | 0.425 | 0.361 | 1.44 |
| Counter increment | Node.js + Redis | 2000 | 16 | 43.19 | 46,310.05 | 0.341 | 0.326 | 0.815 |
| Set add | Node.js + Redis | 2000 | 16 | 40.79 | 49,032.82 | 0.325 | 0.304 | 0.472 |
| Pub/Sub publish | Node.js + Redis | 2000 | 16 | 37.41 | 53,462.63 | 0.297 | 0.292 | 0.408 |
| KV write | Node.js + PostgreSQL | 2000 | 16 | 341.15 | 5,862.61 | 2.73 | 2.09 | 7.48 |
| KV write (batch) | Node.js + PostgreSQL | 2000 | 16 | 49.39 | 40,493.15 | 6.02 | 5.59 | 15.59 |
| KV read | Node.js + PostgreSQL | 2000 | 16 | 245.16 | 8,157.99 | 1.96 | 1.76 | 4.37 |
| KV read (batch) | Node.js + PostgreSQL | 2000 | 16 | 21.7 | 92,181.88 | 2.64 | 2.35 | 5.09 |
| KV read (hot cache) | Node.js + PostgreSQL | 2000 | 16 | 222.23 | 8,999.57 | 1.77 | 1.61 | 3.70 |
| KV read (99% L1) | Node.js + PostgreSQL | 2000 | 16 | 221.17 | 9,042.84 | 1.77 | 1.60 | 3.87 |
| KV read (95% L1) | Node.js + PostgreSQL | 2000 | 16 | 209.51 | 9,545.94 | 1.67 | 1.55 | 3.45 |
| KV read (90% L1) | Node.js + PostgreSQL | 2000 | 16 | 229.2 | 8,725.96 | 1.83 | 1.60 | 5.26 |
| Counter increment | Node.js + PostgreSQL | 2000 | 16 | 211.36 | 9,462.41 | 1.69 | 1.53 | 4.21 |
| Set add | Node.js + PostgreSQL | 2000 | 16 | 491.76 | 4,067.02 | 3.82 | 2.48 | 36.37 |
| Pub/Sub publish | Node.js + PostgreSQL | 2000 | 16 | 162.72 | 12,290.79 | 1.30 | 1.13 | 2.57 |
| KV read (hot cache) | Node.js + PostgreSQL (L1) | 2000 | 16 | 2.53 | 792,029.65 | 0.020 | 0.017 | 0.050 |
| KV read (99% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 4.34 | 460,728.85 | 0.031 | 0.004 | 0.743 |
| KV read (95% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 9.57 | 208,975.5 | 0.075 | 0.001 | 2.09 |
| KV read (90% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 11.48 | 174,288.31 | 0.091 | 0.001 | 2.64 |
| KV write | Bun.js + PostgreSQL | 2000 | 16 | 153.62 | 13,019.03 | 1.22 | 1.05 | 3.49 |
| KV write (batch) | Bun.js + PostgreSQL | 2000 | 16 | 34.03 | 58,776.75 | 4.08 | 4.11 | 8.08 |
| KV read | Bun.js + PostgreSQL | 2000 | 16 | 110.95 | 18,026.22 | 0.883 | 0.771 | 2.57 |
| KV read (batch) | Bun.js + PostgreSQL | 2000 | 16 | 16.04 | 124,657.39 | 1.90 | 1.67 | 5.09 |
| KV read (hot cache) | Bun.js + PostgreSQL | 2000 | 16 | 96.72 | 20,677.9 | 0.770 | 0.721 | 1.87 |
| KV read (99% L1) | Bun.js + PostgreSQL | 2000 | 16 | 106.12 | 18,845.99 | 0.845 | 0.778 | 1.98 |
| KV read (95% L1) | Bun.js + PostgreSQL | 2000 | 16 | 103.44 | 19,334.8 | 0.827 | 0.740 | 1.96 |
| KV read (90% L1) | Bun.js + PostgreSQL | 2000 | 16 | 101.05 | 19,791.92 | 0.806 | 0.740 | 2.00 |
| Counter increment | Bun.js + PostgreSQL | 2000 | 16 | 175.68 | 11,384.53 | 1.40 | 1.16 | 4.72 |
| Set add | Bun.js + PostgreSQL | 2000 | 16 | 311.39 | 6,422.89 | 2.48 | 1.76 | 28.04 |
| Pub/Sub publish | Bun.js + PostgreSQL | 2000 | 16 | 118.8 | 16,834.9 | 0.946 | 0.887 | 2.10 |
| KV read (hot cache) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 3.65 | 548,322.12 | 0.028 | 0.025 | 0.087 |
| KV read (99% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 4.47 | 447,818.61 | 0.034 | 0.008 | 0.338 |
| KV read (95% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 7.78 | 256,985.97 | 0.060 | 0.001 | 1.80 |
| KV read (90% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 8.25 | 242,428.44 | 0.063 | 0.001 | 1.58 |

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
