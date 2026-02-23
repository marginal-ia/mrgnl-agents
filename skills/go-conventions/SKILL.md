---
name: go-conventions
description: Shared Go conventions for all Marginalia repos
---

# Go Conventions — Marginalia

These conventions apply to ALL `mrgnl-*` repositories.

## Formatting and Linting

- Format with `gofmt` (enforced by CI)
- Lint with `golangci-lint run`

## Error Handling

- Wrap errors with context: `fmt.Errorf("create prompt: %w", err)`
- Never use `log.Fatal` or `os.Exit` in library code — only in `main()`
- Prefer `errors.Is()` and `errors.As()` over string matching
- Return `error` as the last return value

## Function Signatures

- Use `context.Context` as the first parameter for all public functions
- Return `error` as the last return value

## Testing

- Table-driven tests for all non-trivial functions
- Use `testify` for assertions (`assert`, `require`)
- Run with race detector: `go test -race ./...`
- Name test functions `TestFunctionName_Scenario`
- Use `t.Helper()` in test helper functions
- Mock external dependencies (DB, S3) in unit tests
- Integration tests connect to real Postgres (via testcontainers or docker-compose)

## Docker

- Multi-stage builds: `golang:1.23` for build, `gcr.io/distroless/static-debian12` for runtime
- Static binaries: `CGO_ENABLED=0`
- Entrypoint is the compiled binary, no shell
- Keep images small (~15 MB per service)

## IDs

- ULID everywhere (generated via `mrgnl-lib/ulid`)
- No UUID, no auto-increment

## Database

- `pgx` driver (native Go PostgreSQL)
- `sqlc` for query generation — write SQL, get type-safe Go
- No ORM
- Strict table ownership: only the owning service writes to its tables

## Commits

Conventional Commits format: `<type>(<scope>): <description>`

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`

Version bumps:
- `fix:` → PATCH (0.0.x)
- `feat:` → MINOR (0.x.0)
- `BREAKING CHANGE:` footer or `!` after type → MAJOR (x.0.0)

## gRPC Ports

| Service | Port |
|---------|------|
| mrgnl-authsvc | 9001 |
| mrgnl-promptsvc | 9002 |
| mrgnl-scrapsvc | 9003 |
| mrgnl-chatsvc | 9004 |
| mrgnl-mcp (HTTP) | 8080 |
