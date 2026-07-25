# Database Performance

Optimize from measurements: inspect `db.Stats()` or `pgxpool.Stat()`, capture query latency, and use `EXPLAIN (ANALYZE, BUFFERS)` against representative data. Do not tune pool sizes or indexes from rules of thumb alone.

## Connection Pools

For `database/sql`:

```go
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(1 * time.Minute)

stats := db.Stats()
slog.InfoContext(ctx, "database pool",
    "open", stats.OpenConnections,
    "in_use", stats.InUse,
    "idle", stats.Idle,
    "wait_count", stats.WaitCount,
    "wait_duration", stats.WaitDuration,
)
```

For native PostgreSQL access, configure `pgxpool.Config` (`MaxConns`, `MinConns`, `MaxConnLifetime`, and `MaxConnIdleTime`) before creating the pool. Keep pool limits below the database and proxy connection budget across all service instances.

Sustained `WaitCount` or pool wait latency means the system is saturated or queries are slow; increasing connections can make database contention worse. Check query latency and database capacity first.

## Batching

Avoid one round trip per row and avoid unbounded batches. Start with a bounded batch, commonly 100-1,000 rows, then benchmark with realistic row sizes and lock contention.

For PostgreSQL bulk ingestion, use `pgx.CopyFrom` when profiling shows the binary `COPY` path is appropriate:

```go
rows := make([][]any, 0, len(users))
for _, u := range users {
    rows = append(rows, []any{u.Name, u.Email})
}

_, err := pool.CopyFrom(ctx,
    pgx.Identifier{"users"},
    []string{"name", "email"},
    pgx.CopyFromRows(rows),
)
if err != nil {
    return fmt.Errorf("copying users: %w", err)
}
```

With `database/sql`, use a prepared statement or multi-row insert inside a bounded transaction, and verify the driver's parameter limit before choosing a batch size.

## Read Performance

- Select only the columns needed by the use case.
- Use a stable, indexed cursor for deep pagination; avoid large `OFFSET` values.
- Prefer `EXISTS` for existence checks instead of counting all matches.
- Eliminate N+1 queries with a join or a batch query.
- Measure rows scanned, returned, latency, and allocation cost.

```sql
-- Cursor pagination; choose the cursor from the actual access path.
SELECT id, created_at, payload
FROM events
WHERE (created_at, id) > ($1, $2)
ORDER BY created_at, id
LIMIT $3;
```

## Indexes and Plans

Inspect existing indexes and query plans before suggesting DDL. Present index additions or removals as reviewed recommendations with table size, write rate, scan counts, and `EXPLAIN` evidence. Never create or drop production indexes automatically.

See `golang-observability` for query and pool metrics. Use the Prometheus server or the project's existing query tooling to inspect PromQL; do not require a separate CLI skill.
