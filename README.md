# mrgnl-agents

Shared Claude Code agents and skills for the [Marginalia](https://github.com/marginal-ia) multi-repo Go microservices platform.

## Installation

```bash
# From the Claude Code marketplace
/plugin marketplace add marginal-ia/mrgnl-agents

# Or for local development
claude --plugin-dir /path/to/mrgnl-agents
```

## Agents

| Agent | Model | Mode | Purpose |
|-------|-------|------|---------|
| **proto-propagator** | opus | default | Proto lifecycle: edit, lint, generate, propagate downstream |
| **service-builder** | inherit | default | Scaffold gRPC handlers, sqlc queries, migrations |
| **cross-repo-coordinator** | opus | default | 6-step coordination workflow, issue tracking |
| **gateway-integrator** | inherit | default | MCP dispatch table, REST routes, SSE in mrgnl-mcp |
| **infra-deployer** | inherit | default | Terraform/CDK for AWS (ECS, RDS, S3, Cloud Map) |
| **frontend-integrator** | inherit | default | React components, API client hooks for mrgnl-web |
| **code-reviewer** | sonnet | plan (read-only) | Review Go, proto, security, cross-repo contracts |
| **test-writer** | inherit | default | Table-driven Go tests, integration tests, bufconn |

## Skills

Skills encode domain knowledge and are loaded automatically by agents (not user-invocable).

| Skill | Used By |
|-------|---------|
| **go-conventions** | service-builder, code-reviewer, test-writer, gateway-integrator |
| **proto-buf-workflow** | proto-propagator, code-reviewer, gateway-integrator |
| **sqlc-patterns** | service-builder, test-writer |
| **grpc-service-patterns** | service-builder, test-writer |
| **cross-repo-protocol** | proto-propagator, cross-repo-coordinator, code-reviewer |
| **docker-build-patterns** | infra-deployer |
| **marginalia-architecture** | proto-propagator, cross-repo-coordinator, infra-deployer, frontend-integrator |
| **mcp-gateway-patterns** | gateway-integrator |

## Hooks

- **Proto file edit**: Reminds to run `buf lint`, `buf breaking`, and `buf generate`
- **sqlc query edit**: Reminds to run `sqlc generate`

## Project Structure

```
mrgnl-agents/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   ├── proto-propagator.md
│   ├── service-builder.md
│   ├── cross-repo-coordinator.md
│   ├── gateway-integrator.md
│   ├── infra-deployer.md
│   ├── frontend-integrator.md
│   ├── code-reviewer.md
│   └── test-writer.md
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
