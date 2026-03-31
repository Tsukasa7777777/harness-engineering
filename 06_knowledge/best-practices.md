# Best Practices for Agent Harness Engineering

**Last Updated**: 2026-03-31

## Overview

These best practices are collected from production agent deployments. They represent patterns that consistently produce better outcomes across different domains and use cases.

## Context Engineering

### 1. Structure Your System Prompt Like Code

Treat the system prompt as source code: version it, review it, test it.

**Do**:
```markdown
# Role
You are a senior backend engineer...

# Rules
## Critical
- Never deploy without approval
## Preferred
- Use existing patterns first
```

**Do not**:
```
You are a helpful assistant. You should try to help with coding questions and you should be careful about security and also you should prefer existing patterns when possible and never deploy without approval.
```

### 2. Front-Load Critical Instructions

Models pay more attention to content at the beginning and end of the context window. Place your most important instructions first.

**Priority ordering**:
1. Identity and role
2. Hard constraints (never/always)
3. Current task
4. Reference material
5. Conversation history

### 3. Use Explicit Formatting

Models follow formatting instructions more reliably when you provide concrete templates rather than abstract descriptions.

**Do**: "Respond using this format: `## Summary\n[summary]\n## Action Items\n- [item]`"

**Do not**: "Organize your response clearly with sections."

### 4. Separate Facts from Instructions

Keep factual reference material separate from behavioral instructions. This prevents the model from confusing "what to know" with "what to do."

## Tool Integration

### 5. Write Tool Descriptions as Contracts

Each tool description should answer: What does it do? When should it be used? When should it NOT be used? What are the required parameters?

```yaml
tool:
  name: "github_create_issue"
  description: "Creates a new issue in a GitHub repository."
  when_to_use: "User explicitly requests creating an issue or tracking a bug."
  when_not_to_use: "User is just discussing a problem — do not create issues unless asked."
  parameters:
    title: "Short, descriptive title (required)"
    body: "Detailed description with context (required)"
    labels: "Relevant labels (optional)"
```

### 6. Implement Confirmation for Destructive Actions

Any tool call that creates, modifies, or deletes external data should require explicit confirmation from the user before execution.

**Low risk** (read-only): Execute immediately  
**Medium risk** (create/update): Confirm before executing  
**High risk** (delete/deploy): Confirm with details of what will happen

### 7. Handle Tool Errors Gracefully

Tools fail. Network errors, rate limits, auth failures, invalid inputs — design for all of them.

```
If tool fails:
  1. Parse the error message
  2. If retryable (network, rate limit): retry with backoff
  3. If not retryable (auth, not found): inform user clearly
  4. Never show raw error traces to the user
  5. Log full error details for debugging
```

### 8. Minimize Tool Calls

Each tool call adds latency, cost, and potential failure points. Before making a tool call, ask: "Can I answer this from context already available?"

## Evaluation

### 9. Eval Early and Often

Do not wait until deployment to evaluate. Run evals:
- During development (every prompt change)
- Before merge (CI/CD gate)
- After deployment (production monitoring)
- On a schedule (weekly regression)

### 10. Test the Failure Cases, Not Just Happy Paths

Your eval suite should include:
- Edge cases (empty input, very long input)
- Adversarial inputs (prompt injection, policy violations)
- Tool failures (network errors, invalid responses)
- Ambiguous requests (requires clarification)

### 11. Track Metrics Over Time

Point-in-time eval scores are useful; trends are more useful. Track:
- Accuracy over time (are we improving?)
- Regression rate per release (are we breaking things?)
- Escalation rate (are we automating enough?)
- User satisfaction (are users happy?)

## Workflow Design

### 12. Start Simple, Add Complexity Only When Needed

Begin with the simplest architecture (single agent + tools) and add complexity only when you have evidence that it is needed.

```
Single Agent -> Not enough context space? -> Add sub-agents
Single Agent -> Too many tools?           -> Add a router
Single Agent -> Quality not high enough?  -> Add iterative refinement
```

### 13. Design for Observability

Every agent action should be logged with:
- What action was taken
- Why it was taken (agent's reasoning)
- What the result was
- How long it took

This makes debugging, auditing, and improvement possible.

### 14. Set Clear Escalation Boundaries

Define exactly when the agent should escalate to a human:
- Confidence below threshold
- Action exceeds authority level
- User explicitly requests human help
- Error count exceeds limit

### 15. Version Everything

Version your:
- System prompts
- Tool configurations
- Eval suites
- Agent personas

This enables rollback, A/B testing, and regression tracking.

## Common Mistakes

| Mistake | Why It Happens | What To Do Instead |
|---------|---------------|-------------------|
| Giant monolithic system prompt | "Just add one more instruction" | Modularize into layered sections |
| No evals | "It seems to work fine" | Build evals from day one |
| Hardcoding model assumptions | "This model always does X" | Test assumptions, design for model changes |
| Ignoring token costs | "It is just an API call" | Monitor and optimize token usage |
| Testing only happy paths | "It works for normal inputs" | Test edge cases and adversarial inputs |
| No fallback strategy | "The tool will always work" | Design graceful degradation for every tool |
| Premature multi-agent | "Multi-agent sounds cool" | Start with one agent, split when needed |
