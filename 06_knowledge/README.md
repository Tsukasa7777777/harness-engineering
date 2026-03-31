# 06_knowledge — Domain Knowledge Base

This section curates domain-specific knowledge that agents reference when making decisions, providing recommendations, or answering questions.

## Contents

| File | Purpose |
|------|---------|
| `architecture-patterns.md` | Common harness architecture patterns and when to use them |
| `failure-modes.md` | Known failure modes in agent systems and mitigations |
| `best-practices.md` | Collected best practices from production agent deployments |

## Purpose

Domain knowledge bridges the gap between a model's general capabilities and the specific expertise required for your use case. Well-curated knowledge:

- Reduces hallucination by providing authoritative reference material
- Improves consistency by establishing shared patterns and terminology
- Accelerates onboarding for new team members working with the harness
- Captures institutional knowledge that would otherwise be lost

## Maintenance

- Review and update knowledge files quarterly
- Add new entries when patterns emerge from production usage
- Remove or update entries that become stale
- Cross-reference with `04_evals/` to validate that knowledge improves eval scores
