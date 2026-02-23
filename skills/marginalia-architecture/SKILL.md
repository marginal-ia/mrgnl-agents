---
name: marginalia-architecture
description: Platform overview, dependency graph, transport layers, ports, AWS topology
---

# Marginalia Architecture

## Overview

Marginalia is a cloud-native SaaS for AI prompt and knowledge management, built as Go microservices with gRPC interfaces. Proto definitions are the single source of truth — REST is auto-generated via gRPC-Gateway, and the MCP adapter translates JSON-RPC tool calls to gRPC.

**GitHub org**: `marginal-ia`
**API domain**: `api.marginalia.sh`
**Transport**: gRPC (internal), REST + MCP + SSE (external via gateway)

## Architecture Diagram

```
Internet → ALB → mrgnl-mcp (gateway, :8080)
                    ├─ gRPC → mrgnl-authsvc   (:9001)
                    ├─ gRPC → mrgnl-promptsvc (:9002)
                    ├─ gRPC → mrgnl-scrapsvc  (:9003)
                    └─ gRPC → mrgnl-chatsvc   (:9004)
                              ↕
                         RDS PostgreSQL + S3
```

## Dependency Graph

```
mrgnl-proto (Go module) — no deps
    ↑
mrgnl-lib (Go module) — depends on: proto
    ↑
mrgnl-promptsvc — depends on: proto, lib │ gRPC: PromptService :9002
mrgnl-scrapsvc  — depends on: proto, lib │ gRPC: ScrapService  :9003
mrgnl-chatsvc   — depends on: proto, lib │ gRPC: ChatService   :9004
mrgnl-authsvc   — depends on: proto, lib │ gRPC: AuthService   :9001

mrgnl-mcp (gateway) — depends on: proto │ HTTP :8080 → gRPC to all services
mrgnl-web (SPA)     — depends on: REST API via gateway
mrgnl-infra (IaC)   — depends on: all service container images
mrgnl-specs (docs)  — no runtime deps
```

## Repositories

| Repo | Type | Language | Purpose |
|------|------|----------|---------|
| mrgnl-proto | module | Go | Protobuf definitions + generated Go code + gRPC-Gateway stubs |
| mrgnl-lib | module | Go | Shared library: DB models (sqlc), S3 client, auth helpers, ULID |
| mrgnl-authsvc | service | Go | AuthService: API keys, OAuth, user/workspace management |
| mrgnl-promptsvc | service | Go | PromptService: prompt CRUD, search, auto-capture |
| mrgnl-scrapsvc | service | Go | ScrapService: scrap CRUD, lifecycle, promotion, injection |
| mrgnl-chatsvc | service | Go | ChatService: channels, messages, SSE, session identity |
| mrgnl-mcp | gateway | Go | API gateway: MCP adapter + gRPC-Gateway (REST) + SSE proxy |
| mrgnl-web | frontend | TypeScript | Dashboard SPA (React) |
| mrgnl-infra | IaC | HCL | Terraform/CDK for AWS deployment |
| mrgnl-specs | docs | Markdown | Product specifications, architecture docs |
| mrgnl-claude | coordination | Markdown | Cross-repo coordination hub |

## Transport Layers

| Path | Protocol | Handler |
|------|----------|---------|
| `/mcp` | Streamable HTTP (JSON-RPC 2.0) | MCP Adapter |
| `/api/v1/*` | REST (HTTP/JSON) | gRPC-Gateway |
| `/sse` | Server-Sent Events | SSE Proxy → chatsvc |

## Port Assignments

| Service | Port | Protocol |
|---------|------|----------|
| mrgnl-authsvc | 9001 | gRPC |
| mrgnl-promptsvc | 9002 | gRPC |
| mrgnl-scrapsvc | 9003 | gRPC |
| mrgnl-chatsvc | 9004 | gRPC |
| mrgnl-mcp (gateway) | 8080 | HTTP |
| PostgreSQL | 5432 | TCP |
| LocalStack (dev) | 4566 | HTTP |

## AWS Topology

- **ECS Fargate**: all services + gateway
- **RDS PostgreSQL 16**: shared database
- **S3**: content storage (marginalia-content) + web assets
- **ALB**: TLS termination, path-based routing
- **CloudFront**: web dashboard (mrgnl-web)
- **Cloud Map**: service discovery (*.marginalia.local)
- **Route 53**: api.marginalia.sh
- **Secrets Manager**: DB credentials, signing keys

## Cloud Map Service Discovery

| Service | DNS Name |
|---------|----------|
| mrgnl-authsvc | authsvc.marginalia.local |
| mrgnl-promptsvc | promptsvc.marginalia.local |
| mrgnl-scrapsvc | scrapsvc.marginalia.local |
| mrgnl-chatsvc | chatsvc.marginalia.local |

## DB Table Ownership

| Service | Tables |
|---------|--------|
| mrgnl-authsvc | `user`, `workspace`, `membership`, `api_key` |
| mrgnl-promptsvc | `prompt`, `session` (auto-capture) |
| mrgnl-scrapsvc | `scrap` |
| mrgnl-chatsvc | `channel`, `message`, `session` (chat identity) |

Write ownership is strict — only the owning service writes to its tables.

## Cross-Repo Contract Boundaries

| Change in... | Affects... |
|-------------|-----------|
| `mrgnl-proto` (any .proto file) | ALL service repos + gateway |
| `mrgnl-lib` (any exported function) | All 4 service repos (not gateway) |
| Service gRPC API (new/changed RPC) | `mrgnl-mcp` gateway dispatch table |
| DB schema (table owned by service X) | Service X + any service that reads those tables |
| Port assignment | `mrgnl-mcp` env vars + `mrgnl-infra` config |
