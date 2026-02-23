---
name: proto-buf-workflow
description: Protobuf conventions, buf toolchain, and REST annotation patterns
---

# Proto / Buf Workflow — Marginalia

## Proto File Structure

All service interfaces defined in `mrgnl-proto/proto/marginalia/v1/`:

```
proto/marginalia/v1/
├── common.proto          # Pagination, Visibility, shared types
├── prompt_service.proto  # PromptService RPCs
├── scrap_service.proto   # ScrapService RPCs
├── chat_service.proto    # ChatService RPCs
└── auth_service.proto    # AuthService RPCs
```

Generated Go code: `gen/marginalia/v1/*.go`

## Buf Commands

```bash
# Lint proto files
buf lint

# Check for breaking changes against main
buf breaking --against .git#branch=main

# Generate Go code from protos
buf generate
```

Always run in this order: lint → breaking → generate.

## Field Number Rules

- Field numbers are permanent — never reuse a deleted field number
- Use `reserved` for removed fields:
  ```protobuf
  message Prompt {
    reserved 7;          // was: deprecated_field
    reserved "old_name"; // reserve the name too
  }
  ```

## REST Mapping with google.api.http

Every RPC that needs a REST endpoint must have an `http` annotation:

```protobuf
import "google/api/annotations.proto";

service PromptService {
  rpc CreatePrompt(CreatePromptRequest) returns (CreatePromptResponse) {
    option (google.api.http) = {
      post: "/api/v1/prompts"
      body: "*"
    };
  }

  rpc GetPrompt(GetPromptRequest) returns (GetPromptResponse) {
    option (google.api.http) = {
      get: "/api/v1/prompts/{id}"
    };
  }

  rpc ListPrompts(ListPromptsRequest) returns (ListPromptsResponse) {
    option (google.api.http) = {
      get: "/api/v1/prompts"
    };
  }

  rpc UpdatePrompt(UpdatePromptRequest) returns (UpdatePromptResponse) {
    option (google.api.http) = {
      patch: "/api/v1/prompts/{id}"
      body: "*"
    };
  }

  rpc DeletePrompt(DeletePromptRequest) returns (DeletePromptResponse) {
    option (google.api.http) = {
      delete: "/api/v1/prompts/{id}"
    };
  }
}
```

## Breaking Change Detection

A change is breaking if it:
- Removes or renames a field
- Changes a field's type
- Changes a field number
- Removes or renames an RPC
- Changes an RPC's request/response type

A change is NOT breaking if it:
- Adds a new field (with a new field number)
- Adds a new RPC
- Adds a new message type

## Workflow for Proto Changes

1. Edit the `.proto` file
2. Run `buf lint` — fix any lint issues
3. Run `buf breaking --against .git#branch=main` — if breaking, file coordination issue
4. Run `buf generate` — regenerate Go code
5. Commit both proto and generated files together
6. Use conventional commit: `feat(proto): add SearchPrompts filter field`
7. If breaking: `feat!(proto): rename PromptText to Content`

## Impact

Changes to `mrgnl-proto` affect:
- ALL 4 service repos (mrgnl-authsvc, mrgnl-promptsvc, mrgnl-scrapsvc, mrgnl-chatsvc)
- The gateway (mrgnl-mcp)

Always check COMPONENT_REGISTRY.md for the full dependency graph.
