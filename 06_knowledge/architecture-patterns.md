# Architecture Patterns

**Last Updated**: 2026-03-31

## Overview

This document catalogs proven architecture patterns for agent harnesses, with guidance on when to use each pattern and common pitfalls to avoid.

## Pattern Catalog

### 1. Single Agent with Tools

**Description**: One agent with direct access to multiple tools via MCP.

```
User -> Agent -> Tool A
                 Tool B
                 Tool C
```

**When to use**:
- Tasks are straightforward and well-defined
- Tool set is small (< 10 tools)
- No need for specialized sub-agents
- Latency requirements are strict

**Strengths**:
- Simplest to implement and debug
- Lowest latency (no agent-to-agent overhead)
- Easiest to evaluate and monitor

**Pitfalls**:
- Context window fills up quickly with many tool descriptions
- Agent may struggle to select the right tool as tool count grows
- No isolation between different capability domains

---

### 2. Orchestrator-Worker

**Description**: A primary agent delegates subtasks to specialized worker agents.

```
User -> Orchestrator -> Worker A (research)
                        Worker B (coding)
                        Worker C (analysis)
```

**When to use**:
- Tasks require multiple distinct expertise areas
- Individual subtasks can be clearly defined
- You need parallelism (workers can run concurrently)
- Context window pressure on single agent is too high

**Strengths**:
- Each worker has a focused context window
- Workers can be independently optimized and evaluated
- Parallel execution reduces total latency for complex tasks

**Pitfalls**:
- Orchestrator overhead adds latency and cost
- Information loss between orchestrator and workers
- Harder to debug when things go wrong
- Workers may produce conflicting outputs

---

### 3. Pipeline (Sequential)

**Description**: Agents process data in a fixed sequence, each adding value.

```
User -> Agent A (extract) -> Agent B (transform) -> Agent C (format) -> Output
```

**When to use**:
- Task has clear, sequential stages
- Each stage has different requirements or expertise
- Output of one stage is input to the next
- Quality gates needed between stages

**Strengths**:
- Clear responsibility boundaries
- Easy to insert quality checks between stages
- Each stage can use a different model or configuration

**Pitfalls**:
- Total latency is the sum of all stages
- Error in early stages propagates through the pipeline
- Inflexible — hard to handle tasks that do not fit the pipeline

---

### 4. Iterative Refinement

**Description**: A single agent (or agent pair) repeatedly improves output through multiple passes.

```
User -> Agent (draft) -> Evaluator -> Agent (refine) -> Evaluator -> ... -> Output
```

**When to use**:
- Output quality is critical and worth extra latency
- Clear evaluation criteria exist for measuring improvement
- Tasks involve creative or complex generation
- Diminishing returns are acceptable (each pass improves less)

**Strengths**:
- Converges toward high quality
- Self-correcting (catches its own errors)
- Works well for writing, code generation, analysis

**Pitfalls**:
- Can loop indefinitely without convergence criteria
- Expensive (multiple model calls per task)
- May over-optimize for evaluation criteria at expense of actual quality

---

### 5. Router

**Description**: A lightweight agent classifies the request and routes to the appropriate specialist.

```
User -> Router -> Specialist A (if type = A)
                  Specialist B (if type = B)
                  Specialist C (if type = C)
                  Fallback (if unclassified)
```

**When to use**:
- Requests fall into distinct categories
- Each category requires different tools, context, or expertise
- You want to minimize unnecessary context loading
- High request volume with diverse request types

**Strengths**:
- Each specialist has optimized context for its domain
- Efficient resource usage (only load what is needed)
- Easy to add new specialists

**Pitfalls**:
- Router misclassification sends requests to wrong specialist
- Requests that span multiple categories are problematic
- Need a robust fallback for unclassifiable requests

---

### 6. Consensus (Multi-Agent Voting)

**Description**: Multiple agents independently process the same input, and a consensus mechanism selects the best output.

```
User -> Agent A -> +
        Agent B -> +-> Consensus -> Output
        Agent C -> +
```

**When to use**:
- Correctness is paramount (medical, legal, financial)
- Task has objectively verifiable outputs
- Cost is less important than accuracy
- You need confidence scores

**Strengths**:
- Higher accuracy through diversity of approaches
- Natural confidence metric (agreement level)
- Resilient to individual agent failures

**Pitfalls**:
- 3x or more cost multiplier
- Latency limited by slowest agent
- Consensus mechanism can be complex to implement well
- May average toward mediocrity on creative tasks

## Choosing a Pattern

```
Is the task simple with < 10 tools?
  -> YES: Single Agent with Tools
  -> NO: Continue

Can the task be decomposed into sequential stages?
  -> YES: Pipeline
  -> NO: Continue

Does the task require multiple expertise areas simultaneously?
  -> YES: Orchestrator-Worker
  -> NO: Continue

Do requests fall into distinct categories?
  -> YES: Router
  -> NO: Continue

Is output quality critical enough to justify multiple passes?
  -> YES: Iterative Refinement
  -> NO: Continue

Is correctness critical enough to justify multiple agents?
  -> YES: Consensus
  -> NO: Start with Single Agent and evolve
```

## Composing Patterns

Patterns can be combined:

- **Router + Pipeline**: Route to different pipelines based on request type
- **Orchestrator + Iterative**: Workers use iterative refinement internally
- **Pipeline + Consensus**: Each pipeline stage uses voting for accuracy
