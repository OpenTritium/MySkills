# Integration Testing

## Docker Compose Fixture

Create `pkg/myfeature/testdata/docker-compose.yml` for test services:

```yaml
version: "3.8"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    ports:
      - "5433:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6380:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
```

## SQL Schema Fixture

Create `pkg/myfeature/testdata/schema.sql` for database initialization:

```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Test Data Fixture

Create `pkg/myfeature/testdata/testdata.sql`:

```sql
INSERT INTO users (name, email) VALUES
    ('Alice Johnson', 'alice@example.com'),
    ('Bob Smith', 'bob@example.com'),
    ('Charlie Brown', 'charlie@example.com');

INSERT INTO orders (user_id, amount, status) VALUES
    (1, 100.00, 'completed'),
    (1, 50.00, 'pending'),
    (2, 200.00, 'completed');
```

## Using Fixtures in Tests

Use the standard `testing` package for new integration tests. A Testify suite is reasonable only when the repository already uses that dependency.

```go
//go:build integration

package database_test

import (
    "database/sql"
    "math"
    "os"
    "os/exec"
    "testing"
    "time"
)

func TestDatabaseFixtures(t *testing.T) {
    cmd := exec.Command("docker-compose", "-f", "testdata/docker-compose.yml", "up", "-d")
    if err := cmd.Run(); err != nil {
        t.Fatalf("failed to start docker-compose: %v", err)
    }
    t.Cleanup(func() {
        _ = exec.Command("docker-compose", "-f", "testdata/docker-compose.yml", "down", "-v").Run()
    })

    time.Sleep(5 * time.Second)

    db, err := sql.Open("postgres", "postgres://test:test@localhost:5433/testdb?sslmode=disable")
    if err != nil {
        t.Fatalf("failed to connect to database: %v", err)
    }
    t.Cleanup(func() { _ = db.Close() })

    schema, err := os.ReadFile("testdata/schema.sql")
    if err != nil {
        t.Fatal(err)
    }
    _, err = db.Exec(string(schema))
    if err != nil {
        t.Fatalf("failed to run schema: %v", err)
    }

    _, err = db.Exec("TRUNCATE TABLE orders, users CASCADE")
    if err != nil {
        t.Fatalf("failed to clear database: %v", err)
    }

    testdata, err := os.ReadFile("testdata/testdata.sql")
    if err != nil {
        t.Fatal(err)
    }
    _, err = db.Exec(string(testdata))
    if err != nil {
        t.Fatalf("failed to load test data: %v", err)
    }

    t.Run("user count", func(t *testing.T) {
        var count int
        if err := db.QueryRow("SELECT COUNT(*) FROM users").Scan(&count); err != nil {
            t.Fatal(err)
        }
        if count != 3 {
            t.Fatalf("user count = %d, want 3", count)
        }
    })

    t.Run("order sum", func(t *testing.T) {
        var sum float64
        if err := db.QueryRow("SELECT SUM(amount) FROM orders").Scan(&sum); err != nil {
            t.Fatal(err)
        }
        if math.Abs(sum-350.0) > 0.01 {
            t.Fatalf("order sum = %v, want 350.0", sum)
        }
    })
}
```

## Test Helper with Embedded Fixtures

```go
package myfeature

import (
    "database/sql"
    "embed"
)

//go:embed testdata/schema.sql testdata/testdata.sql
var fixtures embed.FS

func SetupDB(db *sql.DB) error {
    schema, err := fixtures.ReadFile("testdata/schema.sql")
    if err != nil {
        return err
    }
    if _, err := db.Exec(string(schema)); err != nil {
        return err
    }

    data, err := fixtures.ReadFile("testdata/testdata.sql")
    if err != nil {
        return err
    }
    if _, err := db.Exec(string(data)); err != nil {
        return err
    }
    return nil
}
```
