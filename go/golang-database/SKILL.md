---
name: golang-database
description: "Design and review Go database access with database/sql and PostgreSQL pgx/v5: parameterized queries, scanning, NULL handling, transactions, isolation, pooling, batching, context propagation, and migrations. Use for database tests and query debugging. Does not generate schemas or migration SQL."
license: MIT
---

**Modes:**

- **Write mode** — inspect existing query and repository conventions before adding code; preserve the project's driver and transaction boundary.
- **Review/debug mode** — check parameterization, context propagation, row cleanup, error handling, transaction scope, and pool configuration.

# Go Database Best Practices

Prefer the standard `database/sql` API for portable drivers. For PostgreSQL-specific features or a native pool, use `github.com/jackc/pgx/v5` and `pgxpool`. Keep SQL explicit and avoid ORMs unless the project has already committed to one.

## Rules

1. Parameterize every value; never concatenate user input into SQL.
2. Pass request context to every query, exec, ping, and transaction operation.
3. Handle `sql.ErrNoRows` with `errors.Is`; translate storage errors at the repository boundary.
4. Close `*sql.Rows` immediately after a successful query and check `rows.Err()` after iteration.
5. Use `ExecContext` for statements that do not return rows.
6. Use one transaction for an atomic multi-statement operation; commit only after every step succeeds.
7. Use `SELECT ... FOR UPDATE` only inside the transaction that will modify the selected rows.
8. Configure and measure connection pools; do not guess limits from application concurrency alone.
9. Batch work in bounded chunks and measure query plans with `EXPLAIN (ANALYZE, BUFFERS)` before tuning.
10. Treat schema and migration changes as reviewed artifacts; this skill does not generate them.

## Driver Choice

| Need | Default |
| --- | --- |
| Portable SQL and minimal dependencies | `database/sql` plus the project's driver |
| PostgreSQL native protocol, `COPY`, notifications, arrays, or a native pool | `github.com/jackc/pgx/v5` and `pgxpool` |
| Existing project dependency | Keep it when migration cost exceeds the benefit; do not add a second abstraction |

Do not add `sqlx`, an ORM, or a query builder to a new project solely for convenience. If an existing project already uses one, contain it at the repository boundary and follow its official documentation.

## Parameterized Queries

```go
var user User
err := db.QueryRowContext(ctx,
    `SELECT id, name, email FROM users WHERE email = $1`,
    email,
).Scan(&user.ID, &user.Name, &user.Email)
if err != nil {
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrUserNotFound
    }
    return nil, fmt.Errorf("querying user: %w", err)
}
```

For dynamic identifiers such as `ORDER BY`, map external values to a fixed allowlist. For a dynamic `IN` list, generate placeholders and pass values separately; never join raw values into SQL. PostgreSQL code can often use `= ANY($1)` with a correctly typed array.

See `golang-security` for injection review patterns.

## Scanning and NULL

With `database/sql`, select columns explicitly and scan them in the same order:

```go
rows, err := db.QueryContext(ctx,
    `SELECT id, name, deleted_at FROM users WHERE active = $1`, true,
)
if err != nil {
    return fmt.Errorf("querying users: %w", err)
}
defer rows.Close()

for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name, &u.DeletedAt); err != nil {
        return fmt.Errorf("scanning user: %w", err)
    }
    use(u)
}
if err := rows.Err(); err != nil {
    return fmt.Errorf("iterating users: %w", err)
}
```

Use pointer fields for nullable values when their zero value is meaningful in the domain, or `sql.Null[T]`/driver-specific nullable types when validity must be explicit. See [Struct Scanning](./references/scanning.md).

## Context and Errors

Use `QueryContext`, `QueryRowContext`, `ExecContext`, `BeginTx`, and `PingContext`. Do not create a fresh background context inside repository code. Wrap errors with stable operation context and distinguish cancellation from database failure.

```go
func (r *UserRepository) Get(ctx context.Context, id int64) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx,
        `SELECT id, name FROM users WHERE id = $1`, id,
    ).Scan(&u.ID, &u.Name)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("getting user %d: %w", id, err)
    }
    return &u, nil
}
```

## Transactions

Use `db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})` when the operation requires stronger isolation, and always rollback on every error path. For pgx, use `pool.BeginTx(ctx, pgx.TxOptions{IsoLevel: pgx.Serializable})` and its native transaction methods.

Keep transaction ownership in the service or use-case that defines atomicity. Repositories should accept a narrow executor interface only when the codebase needs the same operation to run against both `*sql.DB` and `*sql.Tx`.

For locking patterns, deadlock ordering, retries, and isolation tradeoffs, read [Transactions](./references/transactions.md).

## Pooling and Batches

For `database/sql`, configure `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`, and `SetConnMaxIdleTime` from database capacity and observed `db.Stats()` wait time. For PostgreSQL `pgxpool`, configure `MaxConns`, `MinConns`, `MaxConnLifetime`, and `MaxConnIdleTime` through `pgxpool.Config`.

Prefer a bounded batch size, explicit columns, and a transaction where the batch must be atomic. PostgreSQL bulk ingestion should use `pgx.CopyFrom` when profiling shows `COPY` is appropriate. See [Database Performance](./references/performance.md).

## Migrations and Schema

Use a reviewed migration tool such as [golang-migrate](https://github.com/golang-migrate/migrate) or [Atlas](https://atlasgo.io/). Do not invent schema DDL from application structs. Indexes, constraints, triggers, views, and row-level security require workload and production-data context.

## Cross-References

- See `golang-security` for SQL injection and secret-handling review.
- See `golang-context` for deadlines and cancellation.
- See `golang-error-handling` for repository error contracts.
- See `golang-testing` for general test strategy.

## References

- [database/sql](https://pkg.go.dev/database/sql)
- [Go database tutorial](https://go.dev/doc/database/)
- [pgx/v5](https://github.com/jackc/pgx)
- [golang-migrate](https://github.com/golang-migrate/migrate)
