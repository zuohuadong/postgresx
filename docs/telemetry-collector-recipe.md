# Telemetry collector recipe

This recipe describes how to use `@postgresx/noredis` around a privacy-conscious
telemetry collector without pushing database or Redis-like dependencies into
client binaries.

## Recommended shape

```text
CLI or desktop client
  -> fetch("POST /telemetry/v1/events")
  -> Bun/Elysia, Hono, or another small HTTPS collector
  -> direct batch INSERT into PostgreSQL
  -> optional pgredis rate limit, cache, outbox, or pg-boss workers
```

Keep the telemetry client thin:

- Use the platform `fetch` API or equivalent HTTP client.
- Do not install `@postgresx/noredis`, `supabase-js`, PostgreSQL drivers, or
  Redis clients in the user-facing telemetry client.
- Do not ship database connection strings, service keys, or queue credentials to
  clients.
- Treat every client payload as untrusted and validate it at the collector
  boundary.

## What pgredis should do

Use `pgredis` as a server-side helper, not as the telemetry ingest pipe itself.

Good fits:

- **Rate limiting** by anonymous install id, IP bucket, build fingerprint, or
  token bucket.
- **Short-lived cache** for collector configuration, sampling rules, public
  build fingerprints, or consent schema metadata.
- **Durable outbox** for post-ingest fan-out such as rollup jobs, webhook
  delivery, or export jobs.
- **`pg-boss` queues** for jobs that need retries, delays, scheduling, and
  explicit failure visibility.

Poor fits:

- Client-side telemetry SDK dependencies.
- Using `LISTEN/NOTIFY` as a durable queue.
- Writing every telemetry event through a Redis-shaped cache abstraction before
  it reaches the append-only event table.
- Storing raw prompts, source code, file paths, repository names, secrets, or
  raw command output in cache keys or queue payloads.

## Minimal Bun collector sketch

```ts
import { SQL } from "bun";
import { createPgredis } from "@postgresx/noredis";
import { createBunSqlAdapter } from "@postgresx/noredis/adapters/bun";

const sql = new SQL(process.env.DATABASE_URL!);
const pg = createPgredis({
  sql: createBunSqlAdapter(sql),
  namespace: "telemetry",
  rateLimit: {
    limit: 120,
    windowMs: 60_000
  },
  outbox: {
    tableName: "telemetry_outbox"
  }
});

await pg.ensureSchema();

export async function ingestTelemetry(request: Request): Promise<Response> {
  const payload = await request.json();
  const installId = String(payload.install_id ?? "missing");

  const limited = await pg.rateLimit!.hit(`install:${installId}`);
  if (!limited.allowed) {
    return Response.json(
      { error: "rate_limited", retry_after_ms: limited.retryAfterMs },
      { status: 429 }
    );
  }

  // Prefer a direct batch INSERT into the telemetry table for accepted events.
  // Keep this table append-only and use explicit allow-listed columns.
  await sql`
    insert into telemetry_events_raw (install_id, event_name, schema_version, payload)
    values (${installId}, ${payload.event_name}, ${payload.schema_version}, ${payload})
  `;

  // Enqueue coarse post-processing work, not the whole ingest hot path.
  await pg.outbox.append("telemetry.rollups", {
    installId,
    eventName: payload.event_name,
    receivedAt: new Date().toISOString()
  });

  return Response.json({ ok: true });
}
```

For real collectors, validate the payload with a schema library, cap request
body size, normalize timestamps on the server, and reject unknown event names.

## Storage guidance

Use application-owned tables for telemetry data:

```sql
CREATE TABLE IF NOT EXISTS telemetry_events_raw (
  id BIGSERIAL PRIMARY KEY,
  received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  install_id TEXT NOT NULL,
  event_name TEXT NOT NULL,
  schema_version INTEGER NOT NULL,
  payload JSONB NOT NULL
);

CREATE INDEX IF NOT EXISTS telemetry_events_raw_received_at_idx
  ON telemetry_events_raw (received_at);

CREATE INDEX IF NOT EXISTS telemetry_events_raw_event_name_idx
  ON telemetry_events_raw (event_name, received_at);
```

Keep `pgredis` tables separate from long-retention telemetry event tables. The
cache and rate-limit tables are operational support tables; raw telemetry,
rollups, and retention policies should belong to the application schema.

## Operational checklist

- Run `pg.cleanupExpired()` or `pg.startCleanupWorker()` for cache and
  rate-limit tables.
- Use `pg.outbox.trim()` or `pg-boss` retention settings for completed jobs.
- Keep `LISTEN/NOTIFY` payloads to identifiers only.
- Monitor event insert latency, rejected payload counts, rate-limit denials,
  outbox pending/locked counts, and worker failures.
- Keep raw event retention short and build aggregated reporting tables for
  long-term metrics.
