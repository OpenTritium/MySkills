# Struct Scanning and NULLable Columns

## `database/sql`

Select columns explicitly and scan them in query order:

```go
type User struct {
    ID        int64
    Name      string
    Email     string
    DeletedAt *time.Time
}

var u User
err := db.QueryRowContext(ctx,
    `SELECT id, name, email, deleted_at FROM users WHERE id = $1`, id,
).Scan(&u.ID, &u.Name, &u.Email, &u.DeletedAt)
```

For multiple rows, close the rows and check the iteration error:

```go
rows, err := db.QueryContext(ctx, `SELECT id, name FROM users`)
if err != nil {
    return err
}
defer rows.Close()

for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name); err != nil {
        return fmt.Errorf("scanning user: %w", err)
    }
    use(u)
}
return rows.Err()
```

## PostgreSQL with pgx/v5

For native pgx queries, `pgx.CollectRows` and `pgx.RowToStructByName` provide generic mapping. Field names must match the selected column names according to pgx's mapping rules; keep an explicit column list.

```go
rows, err := pool.Query(ctx,
    `SELECT id, name, email FROM users WHERE active = $1`, true,
)
if err != nil {
    return nil, fmt.Errorf("querying users: %w", err)
}
users, err := pgx.CollectRows(rows, pgx.RowToStructByName[User])
if err != nil {
    return nil, fmt.Errorf("collecting users: %w", err)
}
return users, nil
```

For unusual aliases or conversions, use `pgx.RowTo` or scan fields explicitly instead of making the mapping implicit.

## NULL Semantics

Choose the representation from the domain contract:

- Pointer fields such as `*string` or `*time.Time` are concise and marshal naturally to JSON `null`.
- `sql.Null[T]` (where supported by the module's Go version) makes validity explicit but may need a JSON adapter.
- Driver-specific nullable types are appropriate when preserving PostgreSQL-specific values.
- `COALESCE` is appropriate only when the query's domain contract intentionally converts NULL to a concrete value.

Do not use `omitempty` to hide a meaningful distinction between NULL and an empty value.
