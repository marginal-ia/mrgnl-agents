# mrgnl-agents

Shared [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) for the [Marginalia](https://github.com/marginal-ia) platform — 8 specialized agents and 8 domain-knowledge skills for working across the multi-repo Go microservices architecture.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed
- Access to the `marginal-ia` GitHub org

## Installation

### From the marketplace (recommended)

Add the marketplace to your workspace settings (`.claude/settings.local.json` or `.claude/settings.json`):

```json
{
  "extraKnownMarketplaces": {
    "mrgnl-agents": {
      "source": {
        "source": "github",
        "repo": "marginal-ia/mrgnl-agents"
      }
    }
  }
}
```

Then enable the plugin:

```json
{
  "enabledPlugins": {
    "mrgnl-agents@mrgnl-agents": true
  }
}
```

### Local development

Point Claude Code at your local checkout:

```bash
claude --plugin-dir /path/to/mrgnl-agents
```

## Usage

Agents are invoked via natural language in Claude Code. Reference them by name:

```
Use the proto-propagator agent to add a new RPC to prompt_service.proto

Use the service-builder agent to implement the SearchPrompts handler

Use the cross-repo-coordinator to check for impacts from the proto change

Use the code-reviewer agent to review my changes before I open a PR
```

You can also reference agents directly in Claude Code with `/agents` to see all available agents.

## Agents

### proto-propagator

**Model**: opus | **Mode**: default | **Memory**: project

Manages the full lifecycle of Protocol Buffer changes — editing `.proto` files, running `buf lint` and `buf breaking`, generating Go code with `buf generate`, and propagating changes to all downstream service repos via the coordination protocol.

### service-builder

**Model**: inherit | **Mode**: default | **Memory**: project

Scaffolds and implements gRPC service handlers, writes sqlc queries, creates database migrations, and wires up server dependencies. Follows the standard Marginalia handler pattern with ULID generation, S3 content splitting, and proper error wrapping.

### cross-repo-coordinator

**Model**: opus | **Mode**: default | **Memory**: project

Runs the 6-step coordination workflow (Detect → Check → Report → Fix → Validate → Close) for changes that span multiple repos. Files GitHub issues in `mrgnl-claude`, tracks progress via `ACTIVE_ISSUES.md`, and ensures nothing falls through the cracks.

### gateway-integrator

**Model**: inherit | **Mode**: default | **Memory**: project

Works on `mrgnl-mcp` — the API gateway. Adds MCP tools to the dispatch table, configures gRPC-Gateway REST routes, manages the SSE proxy, and integrates with the auth middleware.

### infra-deployer

**Model**: inherit | **Mode**: default

Manages Terraform/CDK infrastructure for the AWS deployment: ECS Fargate task definitions, RDS configuration, S3 buckets, ALB rules, Cloud Map service discovery, and Secrets Manager references.

### frontend-integrator

**Model**: inherit | **Mode**: default

Builds React/TypeScript components and typed API client hooks for the `mrgnl-web` dashboard SPA. Handles SSE integration for real-time updates and follows the REST API patterns from the gateway.

### code-reviewer

**Model**: sonnet | **Mode**: plan (read-only)

Performs read-only code reviews. Cannot modify files. Checks Go conventions, proto schema design, security (OWASP top 10), and cross-repo contract boundaries. Outputs structured review with critical issues, suggestions, and coordination impact.

### test-writer

**Model**: inherit | **Mode**: default

Writes table-driven Go tests, gRPC handler tests with bufconn, integration tests with testcontainers, and sqlc query tests. Follows `TestFunctionName_Scenario` naming and uses `testify` assertions.

## Skills

Skills encode domain knowledge and are loaded automatically by the agents that need them. They are not user-invocable.

| Skill | Content | Used By |
|-------|---------|---------|
| **go-conventions** | Formatting, error handling, testing, Docker, commits | service-builder, code-reviewer, test-writer, gateway-integrator |
| **proto-buf-workflow** | Buf toolchain, REST annotations, breaking change rules | proto-propagator, code-reviewer, gateway-integrator |
| **sqlc-patterns** | Query patterns, pgx, ULID, FTS, S3 split, table ownership | service-builder, test-writer |
| **grpc-service-patterns** | Handler signatures, error codes, server setup, Cloud Map | service-builder, test-writer |
| **cross-repo-protocol** | 6-step workflow, dependency graph, impact table, labels | proto-propagator, cross-repo-coordinator, code-reviewer |
| **docker-build-patterns** | Multi-stage Dockerfile, distroless, ECS Fargate deployment | infra-deployer |
| **marginalia-architecture** | Platform overview, all repos, ports, AWS topology | proto-propagator, cross-repo-coordinator, infra-deployer, frontend-integrator |
| **mcp-gateway-patterns** | Dispatch table, REST routes, SSE, auth middleware | gateway-integrator |

## Hooks

Prompt-based reminders that fire after file edits:

| Trigger | Reminder |
|---------|----------|
| Edit a `.proto` file | Run `buf lint && buf breaking --against .git#branch=main`, then `buf generate` |
| Edit a `.sql` file in `queries/` | Run `sqlc generate` |

## Project Structure

```
mrgnl-agents/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest (name, version, author)
├── agents/
│   ├── proto-propagator.md      # Proto changes + downstream propagation
│   ├── service-builder.md       # gRPC handlers, sqlc, migrations
│   ├── cross-repo-coordinator.md # 6-step coordination protocol
│   ├── gateway-integrator.md    # MCP tools + REST routes in mrgnl-mcp
│   ├── infra-deployer.md        # Terraform/CDK for AWS
│   ├── frontend-integrator.md   # React/TS for mrgnl-web
│   ├── code-reviewer.md         # Read-only review (plan mode)
│   └── test-writer.md           # Table-driven Go tests
├── skills/
│   ├── go-conventions/SKILL.md
│   ├── proto-buf-workflow/SKILL.md
│   ├── sqlc-patterns/SKILL.md
│   ├── grpc-service-patterns/SKILL.md
│   ├── cross-repo-protocol/SKILL.md
│   ├── docker-build-patterns/SKILL.md
│   ├── marginalia-architecture/SKILL.md
│   └── mcp-gateway-patterns/SKILL.md
├── hooks/
│   └── hooks.json
└── README.md
```

## Contributing

To modify agents or skills:

1. Clone this repo
2. Edit the relevant `.md` file under `agents/` or `skills/`
3. Test locally: `claude --plugin-dir /path/to/mrgnl-agents`
4. Commit with conventional commits: `feat(agent): improve proto-propagator breaking change detection`
5. Push to `main` — all team members pick up changes on next Claude Code session

### Adding a new agent

Create `agents/<name>.md` with YAML frontmatter:

```yaml
---
model: inherit          # or opus, sonnet
permission: default     # or plan (read-only)
memory: project         # optional, for agents that accumulate knowledge
skills:
  - mrgnl-agents:<skill-name>
---

# Agent Name

Your agent instructions here.
```

### Adding a new skill

Create `skills/<name>/SKILL.md` with YAML frontmatter:

```yaml
---
name: <name>
description: One-line description
user-invocable: false
---

# Skill Title

Skill content here.
```

Then add `mrgnl-agents:<name>` to the `skills` list of any agent that should use it.
