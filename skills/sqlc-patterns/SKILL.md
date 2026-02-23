---
name: sqlc-patterns
description: sqlc query patterns, pgx driver, ULID generation, table ownership, FTS, S3 split
---

# sqlc Patterns — Marginalia

## Overview

Marginalia uses `sqlc` for type-safe query generation with the `pgx` driver. SQL queries live in `mrgnl-lib/db/queries/` and generate Go code.

## Workflow

```bash
# After editing any .sql query file:
sqlc generate

# This produces Go code in mrgnl-lib/db/
```

Always run `sqlc generate` after modifying query files.

## Query File Convention

```sql
-- name: CreatePrompt :one
INSERT INTO prompt (
    id, workspace_id, author_id, prompt_text,
    response_summary, context_json, tags, visibility,
    source, notes, content_ref, created, updated
) VALUES (
    $1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, NOW(), NOW()
) RETURNING *;

-- name: GetPrompt :one
SELECT * FROM prompt WHERE id = $1 AND workspace_id = $2;

-- name: ListPrompts :many
SELECT * FROM prompt
WHERE workspace_id = $1
ORDER BY created DESC
LIMIT $2 OFFSET $3;
```

## ID Generation

All primary keys use ULID (via `mrgnl-lib/ulid`):

```go
import "github.com/marginal-ia/mrgnl-lib/ulid"

id := ulid.New() // returns 26-char Crockford Base32 string
```

- 128-bit, lexicographically sortable
- First 48 bits are millisecond timestamp → naturally time-sorted
- No coordination needed across services

## Table Ownership

Write ownership is strict — only the owning service writes to its tables:

| Service | Tables |
|---------|--------|
| mrgnl-authsvc | `user`, `workspace`, `membership`, `api_key` |
| mrgnl-promptsvc | `prompt`, `session` (auto-capture) |
| mrgnl-scrapsvc | `scrap` |
| mrgnl-chatsvc | `channel`, `message`, `session` (chat identity) |

Cross-service reads are allowed (e.g., prompt service reads `user` for visibility checks).

## Full-Text Search (FTS)

PostgreSQL `tsvector` with weighted columns:

```sql
-- Prompt search index
ALTER TABLE prompt ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(array_to_string(tags, ' '), '')), 'A') ||
    setweight(to_tsvector('english', coalesce(notes, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(prompt_text, '')), 'C')
  ) STORED;

CREATE INDEX idx_prompt_search ON prompt USING gin(search_vector);
```

FTS query pattern:
```sql
-- name: SearchPrompts :many
SELECT * FROM prompt
WHERE workspace_id = $1
  AND search_vector @@ plainto_tsquery('english', $2)
ORDER BY ts_rank(search_vector, plainto_tsquery('english', $2)) DESC
LIMIT $3;
```

## S3 Content Split

When content exceeds 64 KB, store in S3:

| Data | Storage | Condition |
|------|---------|-----------|
| Prompt text < 64 KB | PostgreSQL `prompt_text` | Default |
| Prompt text >= 64 KB | S3 object, key in `content_ref` | `prompt_text` holds preview |
| Scrap content < 64 KB | PostgreSQL `content` | Default |
| Scrap content >= 64 KB | S3 object, key in `content_ref` | `content` holds preview |

S3 object key format: `{workspace_id}/{entity_type}/{entity_id}/content`

## Common Indexes

```sql
-- Workspace-scoped listings
CREATE INDEX idx_prompt_workspace ON prompt(workspace_id, created DESC);
CREATE INDEX idx_prompt_author ON prompt(author_id, created DESC);
CREATE INDEX idx_prompt_tags ON prompt USING gin(tags);

CREATE INDEX idx_scrap_workspace ON scrap(workspace_id, created DESC);
CREATE INDEX idx_scrap_status ON scrap(workspace_id, status, created DESC);
CREATE INDEX idx_scrap_tags ON scrap USING gin(tags);

CREATE INDEX idx_message_channel ON message(channel_id, id);
CREATE INDEX idx_channel_workspace ON channel(workspace_id);
CREATE INDEX idx_session_workspace ON session(workspace_id, last_seen DESC);
```

## Migration Conventions

- Use `golang-migrate` or `atlas`
- File naming: `NNNN_description.up.sql` / `NNNN_description.down.sql`
- Always include both up and down migrations
- Never drop columns in production — mark deprecated first
