# 02_constraints: Constraints, Guardrails & Safe Autonomy

## Purpose

This directory defines the **safety envelope** within which AI agents operate. The goal is to maximize agent autonomy for productivity while preventing catastrophic, irreversible, or unauthorized actions.

The core principle: **an agent should be able to do everything it needs to do, and nothing it should not do.**

## The Autonomy-Safety Spectrum

```
Full Manual Control                                    Full Autonomy
|-------|---------|----------|----------|----------|---------|
        ^         ^          ^          ^          ^
    Prohibited  Approval   Supervised  Autonomous  Unsupervised
    Actions     Required   Autonomy    Actions     (dangerous)
```

We operate in the middle three zones. The two extremes -- full manual control (inefficient) and full unsupervised autonomy (unsafe) -- are both avoided.

### Design Philosophy

1. **Default-deny for destructive actions**: Any action that deletes, publishes, or sends data externally requires explicit permission or pre-authorization via policy.
2. **Default-allow for read and local-write**: Agents should freely read files, search codebases, and write to local workspaces without friction.
3. **Progressive trust**: As confidence in an agent's behavior increases, constraints can be relaxed through explicit policy changes -- never through the agent's self-modification.
4. **Fail-safe over fail-open**: When a constraint is ambiguous, the agent stops and asks rather than guessing.

## Files in This Directory

| File | Purpose |
|------|---------|
| [guardrails.md](./guardrails.md) | Action classification: prohibited, approval-required, and autonomous. Safety rules for file system, network, data, and code execution. |
| [tool-policies.md](./tool-policies.md) | Per-tool policy definitions. Template and examples for scoping what each MCP tool/integration is allowed to do. |
| [sandbox-config.md](./sandbox-config.md) | Runtime containment. File system, network, execution, and resource limit configurations for different environments. |
| [escalation-rules.md](./escalation-rules.md) | Human-in-the-loop triggers. Decision trees for when the agent must stop and consult a human. |

## How These Files Work Together

```
                     +-------------------+
                     |  escalation-rules |
                     | "Should I stop?"  |
                     +--------+----------+
                              |
                    YES: stop  |  NO: proceed
                    and ask    |
                              v
                     +-------------------+
                     |    guardrails     |
                     | "Is this allowed?"|
                     +--------+----------+
                              |
               PROHIBITED     |  ALLOWED (possibly with approval)
               hard-stop      |
                              v
                     +-------------------+
                     |  tool-policies    |
                     | "How do I use     |
                     |  this specific    |
                     |  tool safely?"    |
                     +--------+----------+
                              |
                              v
                     +-------------------+
                     |  sandbox-config   |
                     | "What does the    |
                     |  runtime enforce?"|
                     +-------------------+
```

1. **Escalation rules** are checked first -- they determine if the agent should even attempt the action.
2. **Guardrails** classify the action into prohibited, approval-required, or autonomous.
3. **Tool policies** provide specific rules for the particular tool being used.
4. **Sandbox config** provides the runtime enforcement layer that catches anything the above layers miss.

## Key References

- Anthropic, "Beyond permission prompts: making Claude Code more secure and autonomous" (2025) -- Progressive trust model, `CLAUDE.md` allowlist patterns
- Anthropic, "Writing effective tools for agents" (2025) -- Tool design for safe autonomous operation
- OpenHands, "Mitigating Prompt Injection Attacks" (2025) -- Defense-in-depth for agent tool use
- Thoughtworks, "Assessing internal quality while coding with an agent" (2025) -- Quality gates and verification loops
- Thoughtworks, "Anchoring AI to a reference application" (2025) -- Constraining agent behavior through architectural patterns

## Maintenance

- Review constraints quarterly or after any security incident.
- When adding a new tool/MCP integration, create a corresponding tool policy before enabling it.
- Constraint changes require human review -- agents must never modify files in this directory autonomously.
- Log all escalation events for retrospective analysis.
