# Harness Engineering

> A structured framework for engineering reliable, observable, and well-governed AI agent systems.

## Overview

Harness Engineering is a methodology and reference architecture for building production-grade AI agent harnesses — the orchestration layer that wraps foundation models with context, constraints, tools, evaluations, and workflows.

This repository provides a complete, opinionated structure for teams building AI-powered automation, with a focus on:

- **Reliability** — Guardrails, escalation rules, and sandbox configurations that prevent agent failures from becoming incidents
- **Observability** — Evaluation frameworks and metrics that make agent behavior measurable and improvable
- **Governance** — Tool policies, access controls, and audit trails that satisfy enterprise requirements
- **Composability** — Workflow patterns and runbooks that scale from single-agent tasks to multi-agent orchestration

## Repository Structure

```
harness-engineering/
├── 01_context/          # Agent identity, role definitions, domain knowledge
├── 02_constraints/      # Guardrails, escalation rules, tool policies, sandbox config
├── 03_specs/            # Harness specifications, API contracts, interface definitions
├── 04_evals/            # Evaluation frameworks, benchmarks, quality metrics
├── 05_workflows/        # Orchestration patterns, runbooks, checklists
├── 06_knowledge/        # Domain knowledge bases, reference materials, FAQs
├── 07_tools/            # MCP registry, tool design guides, integration tests
└── LICENSE
```

### Section Details

| Section | Purpose | Key Files |
|---------|---------|-----------|
| **01_context** | Define who the agent is, what it knows, and how it behaves | `agent-identity.md`, `domain-glossary.md`, `persona-templates.md` |
| **02_constraints** | Set boundaries on agent behavior and tool usage | `guardrails.md`, `escalation-rules.md`, `tool-policies.md`, `sandbox-config.md` |
| **03_specs** | Specify harness architecture and interfaces | `harness-spec.md`, `context-window-management.md`, `prompt-architecture.md` |
| **04_evals** | Measure and improve agent quality | `eval-framework.md`, `benchmark-suite.md`, `regression-tests.md` |
| **05_workflows** | Orchestrate agent execution patterns | `patterns/`, `runbooks/`, `12-factor-checklist.md` |
| **06_knowledge** | Curate domain-specific reference material | `architecture-patterns.md`, `failure-modes.md`, `best-practices.md` |
| **07_tools** | Manage tool integrations and MCP servers | `mcp-registry.md`, `tool-design-guide.md`, `integration-tests.md` |

## Getting Started

### For Agent Builders

1. Start with `01_context/agent-identity.md` to define your agent's role and capabilities
2. Set constraints in `02_constraints/` to establish safety boundaries
3. Define your harness spec in `03_specs/harness-spec.md`
4. Build evaluation criteria in `04_evals/eval-framework.md`
5. Implement workflows from `05_workflows/patterns/`

### For Teams Adopting This Framework

1. Fork this repository
2. Customize the context and constraints for your domain
3. Add your tool integrations to `07_tools/mcp-registry.md`
4. Build domain-specific knowledge in `06_knowledge/`
5. Run evaluations from `04_evals/` to validate your harness

## Design Principles

1. **Context is King** — A well-structured context window is the single highest-leverage investment in agent quality
2. **Constraints Enable Freedom** — Clear boundaries let agents operate autonomously with confidence
3. **Evaluate Everything** — If you can't measure it, you can't improve it
4. **Patterns Over Prescriptions** — Provide composable building blocks, not rigid templates
5. **Tools Are Interfaces** — Treat every tool integration as an API contract with versioning and testing

## The 12-Factor Agent Harness

Inspired by the [12-Factor App](https://12factor.net), we define 12 factors for production-grade agent harnesses. See `05_workflows/12-factor-checklist.md` for the complete checklist.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on how to contribute to this project.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

---

Built with the belief that AI agents deserve the same engineering rigor we apply to any production system.
