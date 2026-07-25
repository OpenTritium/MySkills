# Testing Database Code

## Repository and Service Tests

Keep a repository interface when business logic needs a database seam. A small hand-written fake is often clearer than a mocking framework:

```go
type UserRepository interface {
    GetByID(context.Context, int64) (*User, error)
}

type fakeUsers struct {
    user *User
    err  error
}

func (f fakeUsers) GetByID(context.Context, int64) (*User, error) {
    return f.user, f.err
}

func TestUserServiceNotFound(t *testing.T) {
    svc := NewUserService(fakeUsers{err: ErrUserNotFound})

    got, err := svc.GetUser(context.Background(), 999)
    if got != nil {
        t.Fatalf("GetUser() user = %#v, want nil", got)
    }
    if !errors.Is(err, ErrUserNotFound) {
        t.Fatalf("GetUser() error = %v, want ErrUserNotFound", err)
    }
}
```

If a project already depends on `stretchr/testify`, its `assert` and `require` helpers are acceptable in existing tests. Do not add Testify only to avoid a few `testing` checks, and do not make suite-style tests the default.

## Query-Level Tests with sqlmock

Use [DATA-DOG/go-sqlmock](https://github.com/DATA-DOG/go-sqlmock) when a repository uses `database/sql` and a test must verify query shape or driver errors:

```go
func TestGetByID(t *testing.T) {
    db, mock, err := sqlmock.New()
    if err != nil {
        t.Fatal(err)
    }
    defer db.Close()

    mock.ExpectQuery(`SELECT id, name FROM users WHERE id = \$1`).
        WithArgs(int64(1)).
        WillReturnRows(sqlmock.NewRows([]string{"id", "name"]).AddRow(1, "Alice"))

    repo := &userRepository{db: db}
    got, err := repo.GetByID(context.Background(), 1)
    if err != nil {
        t.Fatal(err)
    }
    if got.Name != "Alice" {
        t.Fatalf("GetByID() name = %q, want Alice", got.Name)
    }
    if err := mock.ExpectationsWereMet(); err != nil {
        t.Fatal(err)
    }
}
```

`sqlmock` does not validate SQL against a real schema, planner, or driver. Keep tests for constraints, transaction behavior, and dialect-specific queries at the integration level.

## Integration Tests

Use a build tag when a test requires a real database:

```go
//go:build integration

func TestUserRepository(t *testing.T) {
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        t.Skip("TEST_DATABASE_URL is not set")
    }

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        t.Fatal(err)
    }
    defer db.Close()

    tx, err := db.BeginTx(context.Background(), nil)
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() { _ = tx.Rollback() })

    repo := &userRepository{db: tx}
    // Exercise constraints, scans, transaction behavior, and real SQL here.
    _ = repo
}
```

For PostgreSQL-native repositories, use `pgxpool.New` and `pool.Begin` in the same tagged test. `testcontainers-go` is useful in CI when no shared database is available, but it is an integration-test dependency, not a unit-test default.

Run tagged tests explicitly:

```bash
go test -tags=integration ./internal/repository/...
```

## What to Test

| Behavior | Unit/fake | `sqlmock` | Real database |
| --- | :---: | :---: | :---: |
| Service business rules | yes |  |  |
| Query shape and driver errors |  | yes |  |
| SQL syntax and constraints |  |  | yes |
| Transaction boundaries and locks |  |  | yes |
| NULL round trips and dialect types |  |  | yes |
| Query plan and performance |  |  | yes, with `EXPLAIN` |

Never run database integration tests against production.

See `golang-testing` for test naming, race detection, fuzzing, and CI strategy.
