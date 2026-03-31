# Context Window Management

**Version**: 1.0.0  
**Status**: Draft  
**Last Updated**: 2026-03-31

## Overview

The context window is the most constrained resource in agent systems. Effective management directly impacts response quality, cost efficiency, and system reliability. This document specifies strategies for optimizing context window usage.

## Token Budget Allocation

### Budget Template

```yaml
context_budget:
  total_tokens: 200000          # Model's context window size
  reserved_for_output: 8192     # Max output tokens
  available_for_input: 191808   # Total - reserved

  sections:
    system_identity:
      budget: 2000
      priority: 1               # Highest — never truncated
      description: "Agent role, personality, core constraints"
    
    guardrails:
      budget: 1500
      priority: 2
      description: "Hard constraints and safety rules"
    
    task_instructions:
      budget: 3000
      priority: 3
      description: "Current task context and instructions"
    
    tool_results:
      budget: 50000
      priority: 4
      description: "Results from tool calls"
      truncation: "summarize"   # Strategy when over budget
    
    retrieved_context:
      budget: 80000
      priority: 5
      description: "RAG results, documentation, examples"
      truncation: "drop_oldest"
    
    conversation_history:
      budget: 50000
      priority: 6               # Lowest — first to truncate
      description: "Prior turns in the conversation"
      truncation: "sliding_window"
```

### Priority Rules

1. **Never truncate** priorities 1-2 (identity and guardrails)
2. **Summarize** rather than drop tool results
3. **Use sliding window** for conversation history
4. **Drop oldest first** for retrieved context

## Truncation Strategies

### Sliding Window
Keep the N most recent conversation turns. Always preserve the first turn (which often contains the original task).

```
[Turn 1] [... gap ...] [Turn N-k] ... [Turn N]
```

### Summarization
Replace verbose content with a compressed summary. Useful for tool results and long documents.

```
Before: 15,000 tokens of raw API response
After:  500-token summary of key findings
```

### Priority Dropping
Remove entire sections in reverse priority order when approaching the budget limit.

### Chunking
Split large documents into chunks, include only the most relevant chunks based on semantic similarity to the current query.

## Context Freshness

### Staleness Rules

| Content Type | Max Age | Refresh Strategy |
|-------------|---------|-----------------|
| System identity | Never stale | Static |
| Guardrails | Never stale | Static |
| Tool results | Current turn only | Re-fetch if needed |
| Retrieved docs | 24 hours | Re-retrieve on next query |
| Conversation history | Current session | Summarize at session boundary |

## Monitoring

### Metrics to Track

- **Context utilization**: % of available tokens used per request
- **Truncation events**: How often each section gets truncated
- **Section sizes**: Average token count per section over time
- **Quality correlation**: Eval scores vs. context utilization

### Alerts

- Context utilization > 95% — risk of lost information
- System identity or guardrails truncated — critical safety issue
- Conversation history consuming > 60% of budget — consider summarization

## Best Practices

1. **Measure before optimizing** — Profile your actual context usage before changing budgets
2. **Test truncation impact** — Run evals with simulated truncation to understand quality degradation
3. **Cache aggressively** — Pre-compute and cache context fragments that do not change between requests
4. **Use structured formats** — JSON/YAML is more token-efficient than verbose prose for structured data
5. **Separate concerns** — Keep different types of context in distinct sections for independent budget management
