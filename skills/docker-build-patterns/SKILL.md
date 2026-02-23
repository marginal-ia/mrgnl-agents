---
name: docker-build-patterns
description: Multi-stage Dockerfile patterns, distroless images, CGO_ENABLED=0, ECS Fargate deployment
user-invocable: false
---

# Docker Build Patterns — Marginalia

## Multi-Stage Dockerfile

All Marginalia services use the same Dockerfile pattern:

```dockerfile
# Build stage
FROM golang:1.23 AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /service ./cmd/server

# Runtime stage
FROM gcr.io/distroless/static-debian12

COPY --from=builder /service /service

ENTRYPOINT ["/service"]
```

## Key Constraints

- **CGO_ENABLED=0**: Static binary, no C dependencies
- **distroless base**: No shell, no package manager, minimal attack surface
- **Single binary entrypoint**: The compiled Go binary is the entrypoint
- **Target image size**: ~15 MB per service

## ECS Fargate Deployment

Each service runs as an ECS Fargate task:

```hcl
resource "aws_ecs_task_definition" "promptsvc" {
  family                   = "mrgnl-promptsvc"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 256
  memory                   = 512

  container_definitions = jsonencode([{
    name  = "promptsvc"
    image = "${aws_ecr_repository.promptsvc.repository_url}:latest"
    portMappings = [{
      containerPort = 9002
      protocol      = "tcp"
    }]
    environment = [
      { name = "GRPC_PORT", value = "9002" },
      { name = "S3_BUCKET", value = "marginalia-content" },
    ]
    secrets = [
      { name = "DATABASE_URL", valueFrom = aws_secretsmanager_secret.db_url.arn },
    ]
  }])
}
```

## Container Port Assignments

| Service | Container Port | Protocol |
|---------|---------------|----------|
| mrgnl-authsvc | 9001 | gRPC (TCP) |
| mrgnl-promptsvc | 9002 | gRPC (TCP) |
| mrgnl-scrapsvc | 9003 | gRPC (TCP) |
| mrgnl-chatsvc | 9004 | gRPC (TCP) |
| mrgnl-mcp | 8080 | HTTP (TCP) |

## Build and Push

```bash
# Build
docker build -t mrgnl-promptsvc .

# Tag for ECR
docker tag mrgnl-promptsvc:latest <account>.dkr.ecr.<region>.amazonaws.com/mrgnl-promptsvc:latest

# Push
docker push <account>.dkr.ecr.<region>.amazonaws.com/mrgnl-promptsvc:latest
```

## Local Development

```yaml
# docker-compose.yml pattern
services:
  promptsvc:
    build: ../mrgnl-promptsvc
    ports:
      - "9002:9002"
    environment:
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/marginalia
      GRPC_PORT: "9002"
      AWS_ENDPOINT: http://localstack:4566
      S3_BUCKET: marginalia-content
    depends_on:
      - postgres
      - localstack
```
