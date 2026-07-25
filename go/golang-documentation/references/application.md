# Application Documentation

→ See `golang-cli` skill for CLI application patterns and frameworks.

## CLI Help Text

For CLI applications, `--help` output is the primary documentation. CLI tools MUST have comprehensive `--help` text:

```go
// Keep usage text close to the parser and describe inputs, outputs, and examples.
flag.Usage = func() {
    fmt.Fprintf(flag.CommandLine.Output(), "Usage: mytool [options]\\n\\n")
    flag.PrintDefaults()
    fmt.Fprintln(flag.CommandLine.Output(), "\\nExamples:")
    fmt.Fprintln(flag.CommandLine.Output(), "  mytool --input data.csv --output report.json")
}
```

---

## Configuration Documentation

Configuration SHOULD be documented. Document all configuration sources in the README or a dedicated `docs/configuration.md`:

````markdown
## Configuration

Configuration is loaded in this order (later sources override earlier ones):

1. Default values
2. Configuration file (`~/.config/mytool/config.yaml`)
3. Environment variables
4. Command-line flags

### Environment Variables

| Variable           | Description                | Default | Required |
| ------------------ | -------------------------- | ------- | -------- |
| `MYTOOL_DB_URL`    | Database connection string | —       | Yes      |
| `MYTOOL_LOG_LEVEL` | Log verbosity              | `info`  | No       |
| `MYTOOL_PORT`      | HTTP server port           | `8080`  | No       |
| `MYTOOL_TIMEOUT`   | Request timeout            | `30s`   | No       |

### Configuration File

```yaml
# ~/.config/mytool/config.yaml
database:
  url: postgres://localhost/mydb
  max_connections: 25
server:
  port: 8080
  read_timeout: 30s
logging:
  level: info
  format: json
```
````

---

## Architecture & design decisions

For complex applications, document architectural decisions in `docs/architecture/`:

```
docs/
  architecture/
    0001-use-postgres-as-primary-store.md
    0002-event-driven-architecture.md
    0003-jwt-for-authentication.md
    README.md
```

Each design document follows a standard format:

```markdown
# Use PostgreSQL as Primary Store

## Context

We need a persistent data store that supports...

## Design

We use PostgreSQL because...

## Consequences

- Positive: ACID transactions, rich query language...
- Negative: Operational overhead, connection management...
```

---

## API Documentation

### REST APIs — OpenAPI

Prefer a reviewed OpenAPI 3.x document as the contract. With `ogen`, generate typed Go clients and servers from that document:

```go
//go:generate go run github.com/ogen-go/ogen/cmd/ogen@vX.Y.Z --target internal/api --package api --clean api/openapi.yaml
```

Commit the specification and either the generated package or the reproducible generation policy. Review the generated diff, compile it, and publish the resulting OpenAPI document through the repository's approved documentation UI or API portal.

### Event-Driven — AsyncAPI

For message-based APIs (Kafka, NATS, RabbitMQ), use [AsyncAPI](https://www.asyncapi.com/):

```yaml
asyncapi: "2.6.0"
info:
  title: Order Events
  version: "1.0.0"
channels:
  orders/created:
    publish:
      message:
        payload:
          type: object
          properties:
            orderId:
              type: string
            amount:
              type: number
```

### gRPC — Protobuf

Protobuf files serve as both code contracts and documentation. Add comments to messages and RPCs:

```protobuf
syntax = "proto3";

// UserService manages user accounts.
service UserService {
  // GetUser retrieves a user by their unique identifier.
  // Returns NOT_FOUND if the user does not exist.
  rpc GetUser(GetUserRequest) returns (User);

  // CreateUser registers a new user account.
  // Returns ALREADY_EXISTS if the email is taken.
  rpc CreateUser(CreateUserRequest) returns (User);
}

// User represents a registered user account.
message User {
  // Unique identifier for the user (UUID v4).
  string id = 1;
  // User's display name (1-100 characters).
  string name = 2;
  // User's email address (must be unique across all users).
  string email = 3;
}
```

Use [buf](https://buf.build/) for linting and breaking change detection:

```bash
buf lint
buf breaking --against '.git#branch=main'
```

For REST+gRPC, use [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) to serve both from the same protobuf definition.

### When to Use Each Format

| API Style | Format | Auto-generation |
| --- | --- | --- |
| REST/HTTP with Go handlers | OpenAPI 3.x | ogen or another repository-approved generator |
| REST/HTTP with framework | OpenAPI 3.x | Framework-specific generator or adapter |
| gRPC services | Protobuf | Proto files are the source of truth |
| gRPC + REST gateway | Protobuf + OpenAPI | grpc-gateway generates OpenAPI |
| Message queues / events | AsyncAPI | Manual or code-gen |
| GraphQL | SDL schema | Schema is the docs |
