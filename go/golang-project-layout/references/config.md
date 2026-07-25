# Application Configuration

Keep configuration loading at the application boundary. Parse environment variables, files, and flags into a typed struct, apply a documented precedence order once, validate it, and pass the result to constructors.

## Where Config Lives

```text
myapp/
├── cmd/myapp/
│   └── main.go                # Entry point and process boundary
├── internal/config/
│   ├── config.go              # Typed config and loading
│   └── config_test.go         # Precedence and validation tests
└── configs/
    └── config.example.yaml    # Non-secret example only
```

## Config Struct

Use one typed struct for application configuration. Keep parsing tags local to the chosen file format and validate cross-field rules after all sources are merged:

```go
type Config struct {
    Port     int    `json:"port" yaml:"port"`
    Host     string `json:"host" yaml:"host"`
    LogLevel string `json:"log_level" yaml:"log_level"`
}
```

Document the precedence order, for example flags over environment variables over an explicit file over defaults. Do not read configuration from global state throughout the codebase.

Sensitive values MUST come from environment variables, secret managers, or injected runtime providers. Never commit credentials or put them in example files.
