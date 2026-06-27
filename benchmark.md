# Benchmark

Generated at: 2026-06-27T09:53:25.171Z

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
| KV write | 44,053.93 | 0.306 | 7,801.26 | 1.59 | 0.18x | 18,695.1 | 0.681 | 0.42x |
| KV write (batch) | 167,695.89 | 1.35 | 39,802.95 | 5.47 | 0.24x | 73,701.06 | 3.36 | 0.44x |
| KV read | 58,219.52 | 0.262 | 11,031.97 | 1.27 | 0.19x | 21,667.74 | 0.600 | 0.37x |
| KV read (batch) | 320,406.17 | 0.577 | 109,250.59 | 1.96 | 0.34x | 149,410.51 | 1.25 | 0.47x |
| KV read (hot cache) L1 | 59,916.68 | 0.243 | 1,201,588.74 | 0.012 | 20.05x | 630,190.01 | 0.021 | 10.52x |
| KV read (99% L1) L1 | 64,486.78 | 0.230 | 692,266.45 | 0.003 | 10.74x | 513,121.94 | 0.007 | 7.96x |
| KV read (95% L1) L1 | 67,464.61 | 0.228 | 257,229.04 | 0.001 | 3.81x | 343,540.55 | 0.001 | 5.09x |
| KV read (90% L1) L1 | 53,399.5 | 0.221 | 223,299.72 | 0.001 | 4.18x | 323,771.5 | 0.001 | 6.06x |
| Counter increment | 68,441.9 | 0.227 | 11,759.31 | 1.17 | 0.17x | 16,166.05 | 0.792 | 0.24x |
| Set add | 73,871.65 | 0.194 | 4,247.72 | 2.13 | 0.06x | 6,412.8 | 1.57 | 0.09x |
| Pub/Sub publish | 78,341.94 | 0.199 | 18,767.49 | 0.771 | 0.24x | 25,804.57 | 0.533 | 0.33x |

## L1 Read Cache

These rows isolate pgredis local memory cache behavior. Mixed hit-rate rows include PostgreSQL misses and are closer to real cache-aside usage than the 100% hot-cache row.

| Operation | Redis | Redis p50 ms | Node PG L1 | Node PG L1 p50 ms | Node PG L1/Redis | Bun PG L1 | Bun PG L1 p50 ms | Bun PG L1/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV read (hot cache) | 59,916.68 | 0.243 | 1,201,588.74 | 0.012 | 20.05x | 630,190.01 | 0.021 | 10.52x |
| KV read (99% L1) | 64,486.78 | 0.230 | 692,266.45 | 0.003 | 10.74x | 513,121.94 | 0.007 | 7.96x |
| KV read (95% L1) | 67,464.61 | 0.228 | 257,229.04 | 0.001 | 3.81x | 343,540.55 | 0.001 | 5.09x |
| KV read (90% L1) | 53,399.5 | 0.221 | 223,299.72 | 0.001 | 4.18x | 323,771.5 | 0.001 | 6.06x |

## L2 Backend Path

These rows disable pgredis L1 and measure direct PostgreSQL access. They are useful for fallback sizing and regression tracking, not as the main cache-hit comparison.

| Operation | Redis | Redis p50 ms | Node PG L2 | Node PG L2 p50 ms | Node PG L2/Redis | Bun PG L2 | Bun PG L2 p50 ms | Bun PG L2/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | 44,053.93 | 0.306 | 7,801.26 | 1.59 | 0.18x | 18,695.1 | 0.681 | 0.42x |
| KV write (batch) | 167,695.89 | 1.35 | 39,802.95 | 5.47 | 0.24x | 73,701.06 | 3.36 | 0.44x |
| KV read | 58,219.52 | 0.262 | 11,031.97 | 1.27 | 0.19x | 21,667.74 | 0.600 | 0.37x |
| KV read (batch) | 320,406.17 | 0.577 | 109,250.59 | 1.96 | 0.34x | 149,410.51 | 1.25 | 0.47x |
| KV read (hot cache) | 59,916.68 | 0.243 | 12,085.34 | 1.16 | 0.2x | 29,070.62 | 0.489 | 0.49x |
| KV read (99% L1) | 64,486.78 | 0.230 | 12,497.14 | 1.14 | 0.19x | 26,677.6 | 0.518 | 0.41x |
| KV read (95% L1) | 67,464.61 | 0.228 | 12,629.25 | 1.15 | 0.19x | 28,707.88 | 0.506 | 0.43x |
| KV read (90% L1) | 53,399.5 | 0.221 | 11,647.82 | 1.19 | 0.22x | 29,435.64 | 0.486 | 0.55x |
| Counter increment | 68,441.9 | 0.227 | 11,759.31 | 1.17 | 0.17x | 16,166.05 | 0.792 | 0.24x |
| Set add | 73,871.65 | 0.194 | 4,247.72 | 2.13 | 0.06x | 6,412.8 | 1.57 | 0.09x |
| Pub/Sub publish | 78,341.94 | 0.199 | 18,767.49 | 0.771 | 0.24x | 25,804.57 | 0.533 | 0.33x |

## Details

| Operation | Backend | Iterations | Concurrency | Duration ms | Ops/sec | Avg ms | p50 ms | p99 ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | Node.js + Redis | 2000 | 16 | 45.4 | 44,053.93 | 0.357 | 0.306 | 1.18 |
| KV write (batch) | Node.js + Redis | 2000 | 16 | 11.93 | 167,695.89 | 1.38 | 1.35 | 2.24 |
| KV read | Node.js + Redis | 2000 | 16 | 34.35 | 58,219.52 | 0.273 | 0.262 | 0.543 |
| KV read (batch) | Node.js + Redis | 2000 | 16 | 6.24 | 320,406.17 | 0.737 | 0.577 | 2.02 |
| KV read (hot cache) | Node.js + Redis | 2000 | 16 | 33.38 | 59,916.68 | 0.265 | 0.243 | 0.466 |
| KV read (99% L1) | Node.js + Redis | 2000 | 16 | 31.01 | 64,486.78 | 0.246 | 0.230 | 0.357 |
| KV read (95% L1) | Node.js + Redis | 2000 | 16 | 29.65 | 67,464.61 | 0.236 | 0.228 | 0.419 |
| KV read (90% L1) | Node.js + Redis | 2000 | 16 | 37.45 | 53,399.5 | 0.298 | 0.221 | 2.18 |
| Counter increment | Node.js + Redis | 2000 | 16 | 29.22 | 68,441.9 | 0.230 | 0.227 | 0.430 |
| Set add | Node.js + Redis | 2000 | 16 | 27.07 | 73,871.65 | 0.215 | 0.194 | 0.480 |
| Pub/Sub publish | Node.js + Redis | 2000 | 16 | 25.53 | 78,341.94 | 0.202 | 0.199 | 0.310 |
| KV write | Node.js + PostgreSQL | 2000 | 16 | 256.37 | 7,801.26 | 2.05 | 1.59 | 5.17 |
| KV write (batch) | Node.js + PostgreSQL | 2000 | 16 | 50.25 | 39,802.95 | 5.97 | 5.47 | 14.17 |
| KV read | Node.js + PostgreSQL | 2000 | 16 | 181.29 | 11,031.97 | 1.45 | 1.27 | 3.46 |
| KV read (batch) | Node.js + PostgreSQL | 2000 | 16 | 18.31 | 109,250.59 | 2.24 | 1.96 | 5.79 |
| KV read (hot cache) | Node.js + PostgreSQL | 2000 | 16 | 165.49 | 12,085.34 | 1.32 | 1.16 | 3.43 |
| KV read (99% L1) | Node.js + PostgreSQL | 2000 | 16 | 160.04 | 12,497.14 | 1.28 | 1.14 | 2.89 |
| KV read (95% L1) | Node.js + PostgreSQL | 2000 | 16 | 158.36 | 12,629.25 | 1.26 | 1.15 | 2.73 |
| KV read (90% L1) | Node.js + PostgreSQL | 2000 | 16 | 171.71 | 11,647.82 | 1.37 | 1.19 | 4.11 |
| Counter increment | Node.js + PostgreSQL | 2000 | 16 | 170.08 | 11,759.31 | 1.36 | 1.17 | 5.04 |
| Set add | Node.js + PostgreSQL | 2000 | 16 | 470.84 | 4,247.72 | 3.75 | 2.13 | 42.45 |
| Pub/Sub publish | Node.js + PostgreSQL | 2000 | 16 | 106.57 | 18,767.49 | 0.851 | 0.771 | 1.88 |
| KV read (hot cache) | Node.js + PostgreSQL (L1) | 2000 | 16 | 1.66 | 1,201,588.74 | 0.013 | 0.012 | 0.038 |
| KV read (99% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 2.89 | 692,266.45 | 0.022 | 0.003 | 0.357 |
| KV read (95% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 7.78 | 257,229.04 | 0.061 | 0.001 | 1.93 |
| KV read (90% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 8.96 | 223,299.72 | 0.068 | 0.001 | 1.78 |
| KV write | Bun.js + PostgreSQL | 2000 | 16 | 106.98 | 18,695.1 | 0.850 | 0.681 | 3.87 |
| KV write (batch) | Bun.js + PostgreSQL | 2000 | 16 | 27.14 | 73,701.06 | 3.28 | 3.36 | 7.99 |
| KV read | Bun.js + PostgreSQL | 2000 | 16 | 92.3 | 21,667.74 | 0.734 | 0.600 | 3.02 |
| KV read (batch) | Bun.js + PostgreSQL | 2000 | 16 | 13.39 | 149,410.51 | 1.58 | 1.25 | 4.24 |
| KV read (hot cache) | Bun.js + PostgreSQL | 2000 | 16 | 68.8 | 29,070.62 | 0.548 | 0.489 | 1.42 |
| KV read (99% L1) | Bun.js + PostgreSQL | 2000 | 16 | 74.97 | 26,677.6 | 0.594 | 0.518 | 1.64 |
| KV read (95% L1) | Bun.js + PostgreSQL | 2000 | 16 | 69.67 | 28,707.88 | 0.556 | 0.506 | 1.70 |
| KV read (90% L1) | Bun.js + PostgreSQL | 2000 | 16 | 67.94 | 29,435.64 | 0.542 | 0.486 | 1.49 |
| Counter increment | Bun.js + PostgreSQL | 2000 | 16 | 123.72 | 16,166.05 | 0.985 | 0.792 | 3.67 |
| Set add | Bun.js + PostgreSQL | 2000 | 16 | 311.88 | 6,412.8 | 2.48 | 1.57 | 33.10 |
| Pub/Sub publish | Bun.js + PostgreSQL | 2000 | 16 | 77.51 | 25,804.57 | 0.617 | 0.533 | 1.68 |
| KV read (hot cache) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 3.17 | 630,190.01 | 0.024 | 0.021 | 0.076 |
| KV read (99% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 3.9 | 513,121.94 | 0.030 | 0.007 | 0.164 |
| KV read (95% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 5.82 | 343,540.55 | 0.046 | 0.001 | 1.42 |
| KV read (90% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 6.18 | 323,771.5 | 0.049 | 0.001 | 1.33 |

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
