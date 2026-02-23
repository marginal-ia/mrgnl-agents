---
name: mcp-gateway-patterns
description: MCP dispatch table, REST routes, SSE endpoint, auth middleware, environment variables
---

# MCP Gateway Patterns — Marginalia

## Overview

`mrgnl-mcp` is the unified API gateway for Marginalia. It exposes three endpoints:

| Path | Protocol | Handler |
|------|----------|---------|
| `/mcp` | Streamable HTTP (JSON-RPC 2.0) | MCP Adapter |
| `/api/v1/*` | REST (HTTP/JSON) | gRPC-Gateway |
| `/sse` | Server-Sent Events | SSE Proxy → chatsvc |

The gateway depends on `mrgnl-proto` only (not `mrgnl-lib`).

## MCP Dispatch Table

The MCP adapter translates JSON-RPC tool calls to gRPC:

| Tool prefix | Target service | Port |
|-------------|----------------|------|
| `prompt_*` | mrgnl-promptsvc | 9002 |
| `scrap_*` | mrgnl-scrapsvc | 9003 |
| `chat_*` | mrgnl-chatsvc | 9004 |

### Tool → RPC Mapping

| MCP Tool | REST Endpoint | gRPC RPC |
|----------|--------------|----------|
| `prompt_log` | `POST /prompts` | `PromptService.CreatePrompt` |
| `prompt_search` | `GET /prompts/search` | `PromptService.SearchPrompts` |
| `prompt_list` | `GET /prompts` | `PromptService.ListPrompts` |
| `prompt_get` | `GET /prompts/:id` | `PromptService.GetPrompt` |
| `prompt_update` | `PATCH /prompts/:id` | `PromptService.UpdatePrompt` |
| `prompt_delete` | `DELETE /prompts/:id` | `PromptService.DeletePrompt` |
| `prompt_auto_start` | `POST /prompts/auto-capture/start` | `PromptService.StartAutoCapture` |
| `prompt_auto_stop` | `POST /prompts/auto-capture/stop` | `PromptService.StopAutoCapture` |
| `scrap_create` | `POST /scraps` | `ScrapService.CreateScrap` |
| `scrap_get` | `GET /scraps/:id` | `ScrapService.GetScrap` |
| `scrap_list` | `GET /scraps` | `ScrapService.ListScraps` |
| `scrap_update` | `PATCH /scraps/:id` | `ScrapService.UpdateScrap` |
| `scrap_promote` | `POST /scraps/:id/promote` | `ScrapService.PromoteScrap` |
| `scrap_archive` | `POST /scraps/:id/archive` | `ScrapService.ArchiveScrap` |
| `scrap_inject` | `GET /scraps/:id/inject` | `ScrapService.InjectScrap` |
| `scrap_delete` | `DELETE /scraps/:id` | `ScrapService.DeleteScrap` |
| `chat_send` | `POST /channels/:name/messages` | `ChatService.SendMessage` |
| `chat_read` | `GET /channels/:name/messages` | `ChatService.ReadMessages` |
| `chat_channels` | `GET /channels` | `ChatService.ListChannels` |
| `chat_create_channel` | `POST /channels` | `ChatService.CreateChannel` |
| `chat_pin` | `POST /channels/:name/pin` | `ChatService.PinRef` |
| `chat_set_identity` | `POST /sessions/identity` | `ChatService.SetIdentity` |

## Auth Middleware

Every request is authenticated:

1. Extract `Authorization: Bearer <token>` header
2. Call `AuthService.ValidateKey` on mrgnl-authsvc:9001
3. Cache valid tokens for 30 seconds (TTL)
4. Inject `workspace_id` and `user_id` into gRPC metadata
5. Forward to target service

Token types:
- API key (`lbk_...`) — for MCP clients and programmatic access
- OAuth session JWT — for web dashboard

## SSE Endpoint

`GET /sse?channel=<name>` proxies events from ChatService:

| Event | Description |
|-------|-------------|
| `message` | New message in subscribed channel |
| `pin` | Reference pinned to channel |
| `ping` | Keepalive (every 30s) |

Backed by PostgreSQL `LISTEN/NOTIFY` → gRPC server-streaming → SSE.

## Environment Variables

```
AUTH_SERVICE=authsvc:9001
PROMPT_SERVICE=promptsvc:9002
SCRAP_SERVICE=scrapsvc:9003
CHAT_SERVICE=chatsvc:9004
HTTP_PORT=8080
```

## REST API Patterns

### Pagination
```
GET /api/v1/prompts?limit=20&offset=40
```
Response: `{ "results": [...], "total": 142, "limit": 20, "offset": 40 }`

### Sorting
```
GET /api/v1/prompts?sort=created:asc
```

### Error responses
```json
{
  "error": {
    "code": "not_found",
    "message": "Prompt 01JMX... not found"
  }
}
```

### Rate limiting
- Per API key: 100 requests/minute
- Per workspace: 1000 requests/minute
- Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

## Adding a New MCP Tool

1. Confirm the RPC exists in `mrgnl-proto` with `google.api.http` annotation
2. Add tool entry to the dispatch table
3. Map JSON-RPC params → gRPC request message
4. Map gRPC response → JSON-RPC result
5. Test both `/mcp` (JSON-RPC) and `/api/v1/*` (REST) paths
6. Commit: `feat(gateway): add <tool_name> MCP tool`
