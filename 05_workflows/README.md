# 05 Workflow Design

Workflow design is the core discipline of harness engineering. A "harness" wraps an LLM call with the control flow, state management, guardrails, and tooling needed to produce reliable agent behavior. This section documents the patterns and runbooks that make agent workflows predictable, debuggable, and safe.

## Why Workflow Design Matters

An LLM call alone is a stateless text-completion function. Turning it into a useful agent requires decisions about:

- **Control flow** -- Who decides what happens next: the LLM, the harness, or a combination?
- **State management** -- Where does working memory live, and how is it passed between steps?
- **Error handling** -- What happens when a tool call fails, a timeout fires, or output quality drops?
- **Human oversight** -- When does a human need to approve, review, or redirect?

Getting these decisions wrong leads to brittle agents that fail silently, loop forever, or produce confidently wrong output. Getting them right produces systems that are as reliable as traditional software while retaining the flexibility of natural language reasoning.

## Foundational Principles

Before choosing a pattern, internalize these principles (drawn from the 12 Factor Agents framework and Anthropic's agent design guidance):

1. **The harness owns control flow, not the LLM.** The LLM proposes; the harness disposes. Your code decides what happens after each LLM call.
2. **LLM calls are the expensive, unreliable step.** Minimize the number of calls. Maximize the quality of context going into each call.
3. **Tools are structured output.** A tool call is the LLM producing structured data that your code then acts on. The LLM does not "execute" anything.
4. **State is explicit.** Working state lives in a data structure your code controls, not in a growing message array.
5. **Agents are DAGs, not black boxes.** Every agent can be decomposed into a directed acyclic graph of deterministic steps and LLM-powered decision points.

## Pattern Catalog

| Pattern | When to Use | Complexity | Key Tradeoff |
|---------|-------------|------------|--------------|
| [Sequential](patterns/sequential.md) | Linear, well-understood tasks | Low | Simple but inflexible |
| [Multi-Agent](patterns/multi-agent.md) | Tasks requiring diverse expertise | High | Powerful but complex to debug |
| [Iterative](patterns/iterative.md) | Quality-sensitive tasks with verifiable output | Medium | Robust but slower |
| [Delegation](patterns/delegation.md) | Large-scope tasks that benefit from decomposition | Medium | Scalable but needs good task boundaries |

### Decision Matrix

```
Is the task a fixed sequence of steps?
  YES --> Sequential
  NO  --> Does it require multiple specialized capabilities?
            YES --> Multi-Agent
            NO  --> Is there a way to verify output quality programmatically?
                      YES --> Iterative
                      NO  --> Does the task scope exceed a single context window?
                                YES --> Delegation
                                NO  --> Sequential (start simple, evolve later)
```

## Operational Runbooks

| Runbook | Purpose |
|---------|---------|
| [New Project Setup](runbooks/new-project-setup.md) | Bootstrap harness engineering on a new project |
| [Incident Response](runbooks/incident-response.md) | Respond when an agent produces bad or unexpected output |

## How to Use This Section

1. **Starting a new agent?** Read the [12 Factor Checklist](12-factor-checklist.md) first, then pick a pattern from the catalog.
2. **Debugging a broken agent?** Go to the [Incident Response](runbooks/incident-response.md) runbook.
3. **Onboarding a new project?** Follow the [New Project Setup](runbooks/new-project-setup.md) runbook.
4. **Evolving an existing agent?** Review the patterns to see if a different one fits your current needs better.

## References

- HumanLayer, "12 Factor Agents" (2024)
- Anthropic, "Building effective agents" (2024)
- Anthropic, "How we built our multi-agent research system" (2025)
- LangChain, "Agent Frameworks, Runtimes, and Harnesses" (2025)
- 12-Factor AgentOps methodology
