# 03_specs — Harness Specifications

This section contains the technical specifications that define how the agent harness is architected, how the context window is managed, and how prompts are structured.

## Contents

| File | Purpose |
|------|---------|
| `harness-spec.md` | Core harness architecture specification |
| `context-window-management.md` | Strategies for optimizing context window usage |
| `prompt-architecture.md` | Prompt structure, layering, and formatting standards |

## Design Philosophy

Specifications in this section follow these principles:

1. **Explicit over implicit** — Every architectural decision is documented with rationale
2. **Versioned** — Specs use semantic versioning so changes are trackable
3. **Testable** — Each spec includes acceptance criteria that can be evaluated
4. **Composable** — Specs can be adopted independently or as a complete system
