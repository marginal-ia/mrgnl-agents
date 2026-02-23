---
name: cross-repo-protocol
description: Full coordination protocol, dependency graph, and impact table for cross-repo changes
---

# Cross-Repo Coordination Protocol — Marginalia

## Overview

When a change in one `mrgnl-*` repo affects other repos, follow this 6-step protocol to ensure nothing falls through the cracks.

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

## The 6-Step Workflow

### 1. Detect

Triggers for cross-repo impact:
- Adding, removing, or renaming a proto field or RPC in `mrgnl-proto`
- Changing a public function signature in `mrgnl-lib`
- Changing a gRPC port assignment
- Modifying DB schema for tables read by other services
- Adding a new MCP tool (requires gateway dispatch table update)
- Changing environment variable names used across services

### 2. Check

Before creating a new issue:
1. Read `ACTIVE_ISSUES.md` in `mrgnl-claude`
2. Check GitHub Issues on `marginal-ia/mrgnl-claude` for `affects:<repo>` labels
3. If a matching issue exists, add a comment with new information

### 3. Report

Create a GitHub Issue in `marginal-ia/mrgnl-claude`:

```markdown
## Cross-Repo Impact: [brief description]

**Source repo**: mrgnl-[source]
**Source change**: [PR link or commit hash]
**Change description**: [what changed]

### Affected repos

- [ ] mrgnl-[repo1] — [what needs to change]
- [ ] mrgnl-[repo2] — [what needs to change]

### Details

[Full description of the change and what downstream repos need to do]

### Verification

[How to verify the fix works end-to-end]
```

Labels: `cross-repo`, `affects:<repo>`, optionally `breaking`, `proto-change`, `db-schema`

Add entry to `ACTIVE_ISSUES.md`:
```
- [#<issue>](<url>) — <brief description> — affects: <repo1>, <repo2>
```

### 4. Fix

When picking up a coordination issue:
1. Read the issue for full context
2. Implement required changes
3. Reference in commit: `Ref: marginal-ia/mrgnl-claude#<number>`
4. Check the checkbox for your repo

### 5. Validate

Confirm end-to-end:
1. All affected repo checkboxes checked
2. Integration tests pass
3. No new breaking changes introduced

### 6. Close

1. Close the GitHub Issue with resolution summary
2. Move entry from `ACTIVE_ISSUES.md` to `RESOLVED_ISSUES.md`
3. Update `API_CHANGES.md` if proto/API change

## Impact Reference Table

| Change Type | Check These |
|------------|------------|
| Proto field added/removed | All service repos using that message type + gateway |
| Proto RPC added | Gateway dispatch table + any client calling the RPC |
| `mrgnl-lib` function signature changed | All 4 service repos |
| DB migration in service X | Service X + services that read from X's tables |
| New environment variable | `mrgnl-infra` Terraform/CDK config |
| Port change | Gateway env vars + infra config + docker-compose |

## Cross-Repo Contract Boundaries

| Change in... | Affects... |
|-------------|-----------|
| `mrgnl-proto` (any .proto file) | ALL service repos + gateway |
| `mrgnl-lib` (any exported function) | All 4 service repos (not gateway) |
| Service gRPC API (new/changed RPC) | `mrgnl-mcp` gateway dispatch table |
| DB schema (table owned by service X) | Service X + any service that reads those tables |
| Port assignment | `mrgnl-mcp` env vars + `mrgnl-infra` config |

## Labels

| Label | Description |
|-------|-------------|
| `cross-repo` | Cross-repo coordination issue |
| `breaking` | Breaking change requiring immediate attention |
| `proto-change` | Involves protobuf definition changes |
| `db-schema` | Involves database schema changes |
| `affects:<repo>` | Affects the specified repo |

## Tracking Issue Hygiene

When closing an issue referenced in a parent/milestone issue:
1. Update the parent issue body — change status marker from open to complete
2. Add a comment noting which sub-issue was completed and work now unblocked
