# Benchmark

Generated at: 2026-06-27T09:52:28.617Z

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
| KV write | 42,174.15 | 0.307 | 7,349.17 | 1.62 | 0.17x | 17,866.36 | 0.696 | 0.42x |
| KV write (batch) | 159,767.58 | 1.22 | 45,536.91 | 5.29 | 0.29x | 69,240.94 | 3.33 | 0.43x |
| KV read | 58,957.32 | 0.251 | 10,834.16 | 1.28 | 0.18x | 23,523.05 | 0.587 | 0.4x |
| KV read (batch) | 286,678.14 | 0.616 | 115,759.67 | 1.84 | 0.4x | 132,588.59 | 1.47 | 0.46x |
| KV read (hot cache) L1 | 57,225.6 | 0.243 | 1,167,609.12 | 0.011 | 20.4x | 686,336.85 | 0.019 | 11.99x |
| KV read (99% L1) L1 | 64,803.66 | 0.225 | 641,959.77 | 0.004 | 9.91x | 540,820.01 | 0.007 | 8.35x |
| KV read (95% L1) L1 | 67,243.82 | 0.229 | 274,826.9 | 0.001 | 4.09x | 321,946 | 0.001 | 4.79x |
| KV read (90% L1) L1 | 52,698.63 | 0.234 | 228,737.98 | 0.001 | 4.34x | 307,734.21 | 0.001 | 5.84x |
| Counter increment | 61,501.41 | 0.240 | 11,511.16 | 1.21 | 0.19x | 16,045.98 | 0.801 | 0.26x |
| Set add | 69,225.09 | 0.207 | 4,168.81 | 2.27 | 0.06x | 6,679.98 | 1.57 | 0.1x |
| Pub/Sub publish | 77,011.96 | 0.201 | 17,919.32 | 0.781 | 0.23x | 21,645.35 | 0.528 | 0.28x |

## L1 Read Cache

These rows isolate pgredis local memory cache behavior. Mixed hit-rate rows include PostgreSQL misses and are closer to real cache-aside usage than the 100% hot-cache row.

| Operation | Redis | Redis p50 ms | Node PG L1 | Node PG L1 p50 ms | Node PG L1/Redis | Bun PG L1 | Bun PG L1 p50 ms | Bun PG L1/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV read (hot cache) | 57,225.6 | 0.243 | 1,167,609.12 | 0.011 | 20.4x | 686,336.85 | 0.019 | 11.99x |
| KV read (99% L1) | 64,803.66 | 0.225 | 641,959.77 | 0.004 | 9.91x | 540,820.01 | 0.007 | 8.35x |
| KV read (95% L1) | 67,243.82 | 0.229 | 274,826.9 | 0.001 | 4.09x | 321,946 | 0.001 | 4.79x |
| KV read (90% L1) | 52,698.63 | 0.234 | 228,737.98 | 0.001 | 4.34x | 307,734.21 | 0.001 | 5.84x |

## L2 Backend Path

These rows disable pgredis L1 and measure direct PostgreSQL access. They are useful for fallback sizing and regression tracking, not as the main cache-hit comparison.

| Operation | Redis | Redis p50 ms | Node PG L2 | Node PG L2 p50 ms | Node PG L2/Redis | Bun PG L2 | Bun PG L2 p50 ms | Bun PG L2/Redis |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | 42,174.15 | 0.307 | 7,349.17 | 1.62 | 0.17x | 17,866.36 | 0.696 | 0.42x |
| KV write (batch) | 159,767.58 | 1.22 | 45,536.91 | 5.29 | 0.29x | 69,240.94 | 3.33 | 0.43x |
| KV read | 58,957.32 | 0.251 | 10,834.16 | 1.28 | 0.18x | 23,523.05 | 0.587 | 0.4x |
| KV read (batch) | 286,678.14 | 0.616 | 115,759.67 | 1.84 | 0.4x | 132,588.59 | 1.47 | 0.46x |
| KV read (hot cache) | 57,225.6 | 0.243 | 11,909.18 | 1.21 | 0.21x | 28,495.11 | 0.509 | 0.5x |
| KV read (99% L1) | 64,803.66 | 0.225 | 12,064.92 | 1.20 | 0.19x | 28,877.67 | 0.502 | 0.45x |
| KV read (95% L1) | 67,243.82 | 0.229 | 12,378.05 | 1.18 | 0.18x | 26,816.72 | 0.522 | 0.4x |
| KV read (90% L1) | 52,698.63 | 0.234 | 11,251.24 | 1.26 | 0.21x | 25,751.35 | 0.546 | 0.49x |
| Counter increment | 61,501.41 | 0.240 | 11,511.16 | 1.21 | 0.19x | 16,045.98 | 0.801 | 0.26x |
| Set add | 69,225.09 | 0.207 | 4,168.81 | 2.27 | 0.06x | 6,679.98 | 1.57 | 0.1x |
| Pub/Sub publish | 77,011.96 | 0.201 | 17,919.32 | 0.781 | 0.23x | 21,645.35 | 0.528 | 0.28x |

## Details

| Operation | Backend | Iterations | Concurrency | Duration ms | Ops/sec | Avg ms | p50 ms | p99 ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| KV write | Node.js + Redis | 2000 | 16 | 47.42 | 42,174.15 | 0.373 | 0.307 | 1.18 |
| KV write (batch) | Node.js + Redis | 2000 | 16 | 12.52 | 159,767.58 | 1.47 | 1.22 | 2.73 |
| KV read | Node.js + Redis | 2000 | 16 | 33.92 | 58,957.32 | 0.269 | 0.251 | 0.547 |
| KV read (batch) | Node.js + Redis | 2000 | 16 | 6.98 | 286,678.14 | 0.802 | 0.616 | 2.53 |
| KV read (hot cache) | Node.js + Redis | 2000 | 16 | 34.95 | 57,225.6 | 0.278 | 0.243 | 0.818 |
| KV read (99% L1) | Node.js + Redis | 2000 | 16 | 30.86 | 64,803.66 | 0.244 | 0.225 | 0.489 |
| KV read (95% L1) | Node.js + Redis | 2000 | 16 | 29.74 | 67,243.82 | 0.236 | 0.229 | 0.394 |
| KV read (90% L1) | Node.js + Redis | 2000 | 16 | 37.95 | 52,698.63 | 0.302 | 0.234 | 2.28 |
| Counter increment | Node.js + Redis | 2000 | 16 | 32.52 | 61,501.41 | 0.257 | 0.240 | 0.588 |
| Set add | Node.js + Redis | 2000 | 16 | 28.89 | 69,225.09 | 0.229 | 0.207 | 0.334 |
| Pub/Sub publish | Node.js + Redis | 2000 | 16 | 25.97 | 77,011.96 | 0.206 | 0.201 | 0.321 |
| KV write | Node.js + PostgreSQL | 2000 | 16 | 272.14 | 7,349.17 | 2.17 | 1.62 | 7.30 |
| KV write (batch) | Node.js + PostgreSQL | 2000 | 16 | 43.92 | 45,536.91 | 5.39 | 5.29 | 14.07 |
| KV read | Node.js + PostgreSQL | 2000 | 16 | 184.6 | 10,834.16 | 1.47 | 1.28 | 4.07 |
| KV read (batch) | Node.js + PostgreSQL | 2000 | 16 | 17.28 | 115,759.67 | 2.13 | 1.84 | 4.86 |
| KV read (hot cache) | Node.js + PostgreSQL | 2000 | 16 | 167.94 | 11,909.18 | 1.34 | 1.21 | 2.93 |
| KV read (99% L1) | Node.js + PostgreSQL | 2000 | 16 | 165.77 | 12,064.92 | 1.32 | 1.20 | 2.89 |
| KV read (95% L1) | Node.js + PostgreSQL | 2000 | 16 | 161.58 | 12,378.05 | 1.29 | 1.18 | 2.95 |
| KV read (90% L1) | Node.js + PostgreSQL | 2000 | 16 | 177.76 | 11,251.24 | 1.42 | 1.26 | 3.64 |
| Counter increment | Node.js + PostgreSQL | 2000 | 16 | 173.74 | 11,511.16 | 1.38 | 1.21 | 4.02 |
| Set add | Node.js + PostgreSQL | 2000 | 16 | 479.75 | 4,168.81 | 3.83 | 2.27 | 41.37 |
| Pub/Sub publish | Node.js + PostgreSQL | 2000 | 16 | 111.61 | 17,919.32 | 0.890 | 0.781 | 2.11 |
| KV read (hot cache) | Node.js + PostgreSQL (L1) | 2000 | 16 | 1.71 | 1,167,609.12 | 0.013 | 0.011 | 0.040 |
| KV read (99% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 3.12 | 641,959.77 | 0.022 | 0.004 | 0.570 |
| KV read (95% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 7.28 | 274,826.9 | 0.057 | 0.001 | 1.55 |
| KV read (90% L1) | Node.js + PostgreSQL (L1) | 2000 | 16 | 8.74 | 228,737.98 | 0.069 | 0.001 | 1.99 |
| KV write | Bun.js + PostgreSQL | 2000 | 16 | 111.94 | 17,866.36 | 0.890 | 0.696 | 3.56 |
| KV write (batch) | Bun.js + PostgreSQL | 2000 | 16 | 28.88 | 69,240.94 | 3.47 | 3.33 | 7.75 |
| KV read | Bun.js + PostgreSQL | 2000 | 16 | 85.02 | 23,523.05 | 0.676 | 0.587 | 2.16 |
| KV read (batch) | Bun.js + PostgreSQL | 2000 | 16 | 15.08 | 132,588.59 | 1.80 | 1.47 | 5.74 |
| KV read (hot cache) | Bun.js + PostgreSQL | 2000 | 16 | 70.19 | 28,495.11 | 0.558 | 0.509 | 1.41 |
| KV read (99% L1) | Bun.js + PostgreSQL | 2000 | 16 | 69.26 | 28,877.67 | 0.549 | 0.502 | 1.49 |
| KV read (95% L1) | Bun.js + PostgreSQL | 2000 | 16 | 74.58 | 26,816.72 | 0.596 | 0.522 | 2.26 |
| KV read (90% L1) | Bun.js + PostgreSQL | 2000 | 16 | 77.67 | 25,751.35 | 0.620 | 0.546 | 2.06 |
| Counter increment | Bun.js + PostgreSQL | 2000 | 16 | 124.64 | 16,045.98 | 0.993 | 0.801 | 3.33 |
| Set add | Bun.js + PostgreSQL | 2000 | 16 | 299.4 | 6,679.98 | 2.38 | 1.57 | 30.48 |
| Pub/Sub publish | Bun.js + PostgreSQL | 2000 | 16 | 92.4 | 21,645.35 | 0.736 | 0.528 | 2.33 |
| KV read (hot cache) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 2.91 | 686,336.85 | 0.022 | 0.019 | 0.081 |
| KV read (99% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 3.7 | 540,820.01 | 0.028 | 0.007 | 0.200 |
| KV read (95% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 6.21 | 321,946 | 0.049 | 0.001 | 1.39 |
| KV read (90% L1) | Bun.js + PostgreSQL (L1) | 2000 | 16 | 6.5 | 307,734.21 | 0.050 | 0.001 | 1.42 |

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
