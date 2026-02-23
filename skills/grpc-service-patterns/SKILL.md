---
name: grpc-service-patterns
description: gRPC handler signatures, error codes, server setup, Cloud Map, environment variables
user-invocable: false
---

# gRPC Service Patterns — Marginalia

## Service Architecture

Each Marginalia service follows the same structure:

```
mrgnl-<name>svc/
├── cmd/server/main.go    # Entry point, server setup
├── internal/
│   ├── server/           # gRPC handler implementations
│   ├── config/           # Environment variable parsing
│   └── ...
├── go.mod
├── go.sum
└── Dockerfile
```

## Handler Signature

```go
func (s *Server) CreatePrompt(ctx context.Context, req *pb.CreatePromptRequest) (*pb.CreatePromptResponse, error) {
    // 1. Validate request
    if req.GetPromptText() == "" {
        return nil, status.Error(codes.InvalidArgument, "prompt_text is required")
    }

    // 2. Generate ID
    id := ulid.New()

    // 3. Execute query (sqlc-generated)
    prompt, err := s.queries.CreatePrompt(ctx, db.CreatePromptParams{
        ID:          id,
        WorkspaceID: auth.WorkspaceIDFromContext(ctx),
        AuthorID:    auth.UserIDFromContext(ctx),
        PromptText:  req.GetPromptText(),
        Tags:        req.GetTags(),
        Visibility:  req.GetVisibility().String(),
    })
    if err != nil {
        return nil, status.Errorf(codes.Internal, "create prompt: %v", err)
    }

    // 4. Convert to proto response
    return &pb.CreatePromptResponse{
        Prompt: toPromptProto(prompt),
    }, nil
}
```

## gRPC Error Codes

Map domain errors to gRPC status codes:

| Situation | gRPC Code |
|-----------|-----------|
| Missing required field | `codes.InvalidArgument` |
| Resource not found | `codes.NotFound` |
| Permission denied | `codes.PermissionDenied` |
| Duplicate resource | `codes.AlreadyExists` |
| Internal error | `codes.Internal` |
| Upstream timeout | `codes.DeadlineExceeded` |

Always wrap errors with context:
```go
return nil, status.Errorf(codes.Internal, "create prompt: %v", err)
```

## Server Setup

```go
func main() {
    cfg := config.Load()

    // Database connection
    pool, err := pgxpool.New(context.Background(), cfg.DatabaseURL)
    if err != nil {
        log.Fatalf("connect to database: %v", err)
    }
    defer pool.Close()

    queries := db.New(pool)

    // gRPC server
    srv := grpc.NewServer(
        grpc.UnaryInterceptor(/* interceptors */),
    )
    pb.RegisterPromptServiceServer(srv, server.New(queries, s3Client))

    lis, err := net.Listen("tcp", ":"+cfg.GRPCPort)
    if err != nil {
        log.Fatalf("listen: %v", err)
    }

    log.Printf("PromptService listening on :%s", cfg.GRPCPort)
    if err := srv.Serve(lis); err != nil {
        log.Fatalf("serve: %v", err)
    }
}
```

## Cloud Map Service Discovery

Services register with AWS Cloud Map for discovery:

| Service | DNS Name | Port |
|---------|----------|------|
| mrgnl-authsvc | authsvc.marginalia.local | 9001 |
| mrgnl-promptsvc | promptsvc.marginalia.local | 9002 |
| mrgnl-scrapsvc | scrapsvc.marginalia.local | 9003 |
| mrgnl-chatsvc | chatsvc.marginalia.local | 9004 |

## Environment Variables

Each service requires:

```
DATABASE_URL=postgres://user:pass@host:5432/marginalia
GRPC_PORT=900X
```

Services with S3 access also need:
```
AWS_ENDPOINT=http://localstack:4566   # dev only
S3_BUCKET=marginalia-content
```

## Inter-Service Communication

- Services communicate via gRPC (not HTTP)
- The gateway (`mrgnl-mcp`) is the only HTTP-facing component
- Auth is handled by the gateway calling `AuthService.ValidateKey`
- Services trust the gateway-injected context (workspace ID, user ID)

## Testing gRPC Handlers

Use `bufconn` for in-process gRPC testing (no network):

```go
lis := bufconn.Listen(1024 * 1024)
srv := grpc.NewServer()
pb.RegisterPromptServiceServer(srv, server.New(mockQueries, mockS3))
go srv.Serve(lis)
```
