# Multi-Agent Coordination Pattern

An orchestrator agent decomposes a task and coordinates specialist agents that work on sub-tasks, potentially in parallel. Communication flows through the orchestrator, not directly between specialists.

## When to Use

- The task requires fundamentally different expertise in different phases (e.g., research + writing + code generation).
- Sub-tasks are independent enough to run in parallel.
- A single prompt would be too complex or context-heavy for one agent.
- You want to isolate failures: one specialist failing should not corrupt another's work.

## When NOT to Use

- The task is naturally sequential with tight data dependencies (use Sequential).
- You have fewer than 3 meaningfully distinct sub-tasks (overhead exceeds benefit).
- Context sharing between agents is the bottleneck (consider a single agent with better context management).

## Architecture

```
                    +-------------------+
                    |   Orchestrator    |
                    |   (Planner +      |
                    |    Coordinator)   |
                    +-------------------+
                     /       |        \
                    v        v         v
          +-----------+ +-----------+ +-----------+
          | Specialist| | Specialist| | Specialist|
          |  Agent A  | |  Agent B  | |  Agent C  |
          | (Research)| | (Analysis)| | (Writing) |
          +-----------+ +-----------+ +-----------+
                    \        |        /
                     v       v       v
                    +-------------------+
                    |   Orchestrator    |
                    |   (Aggregator +   |
                    |    Quality Gate)  |
                    +-------------------+
                            |
                            v
                      Final Output
```

### Component Responsibilities

| Component | Role | LLM or Code? |
|-----------|------|---------------|
| **Orchestrator** | Decomposes task, assigns sub-tasks, aggregates results | Both |
| **Planner** (part of orchestrator) | Determines which specialists to invoke and in what order | LLM |
| **Specialist** | Executes a narrowly scoped sub-task with domain expertise | LLM |
| **Aggregator** (part of orchestrator) | Merges specialist outputs into a coherent result | LLM |
| **Quality Gate** (part of orchestrator) | Validates the aggregated result meets requirements | Code |

## Communication Protocol

### Message Format

All inter-agent communication uses a structured envelope:

```python
@dataclass
class AgentMessage:
    sender: str            # agent identifier
    recipient: str         # agent identifier
    task_id: str           # unique task identifier
    message_type: str      # "task_assignment" | "result" | "error" | "clarification"
    content: str           # natural language description
    structured_data: dict  # machine-readable payload
    timestamp: datetime
    parent_message_id: str | None  # for threading
```

### Communication Rules

1. **Star topology**: All communication goes through the orchestrator. Specialists never talk directly to each other.
2. **Explicit contracts**: Each specialist receives a task assignment with defined inputs, expected output format, and success criteria.
3. **Immutable messages**: Messages are logged and never modified after sending.
4. **Timeout enforcement**: The orchestrator enforces deadlines on specialist responses.

## State Sharing

### Shared State Architecture

```python
@dataclass
class MultiAgentState:
    """Global state owned by the orchestrator."""
    task: str
    plan: list[SubTask]
    specialist_assignments: dict[str, SubTask]
    specialist_results: dict[str, AgentResult]
    aggregated_output: str
    quality_check: dict
    status: str  # "planning" | "executing" | "aggregating" | "complete" | "failed"

@dataclass
class SubTask:
    id: str
    description: str
    assigned_to: str
    dependencies: list[str]  # IDs of sub-tasks this depends on
    context: dict            # data the specialist needs
    expected_output: str     # description of expected output format
    status: str
    result: str | None

@dataclass
class AgentResult:
    agent_id: str
    sub_task_id: str
    output: str
    confidence: float  # 0.0-1.0, self-assessed
    metadata: dict
```

### Context Passing Strategy

| Strategy | When to Use | Tradeoff |
|----------|-------------|----------|
| **Full context** | Small tasks, few specialists | Complete but expensive |
| **Relevant slice** | Large tasks, specialists need different data | Efficient but requires good slicing logic |
| **Summary** | Very large context, specialists need overview | Compact but lossy |
| **On-demand** | Unpredictable context needs | Flexible but adds latency |

Recommended default: **Relevant slice**. The orchestrator pre-selects context relevant to each specialist's sub-task.

## Implementation Template

```python
class MultiAgentWorkflow:
    def __init__(self, llm_client, specialists: dict[str, SpecialistConfig]):
        self.llm = llm_client
        self.specialists = specialists

    def run(self, task: str) -> MultiAgentState:
        state = MultiAgentState(task=task)

        # Phase 1: Plan
        state.plan = self.plan(task)
        log(f"Plan: {len(state.plan)} sub-tasks")

        # Phase 2: Execute (respecting dependencies)
        execution_order = topological_sort(state.plan)
        for batch in execution_order:
            # Run independent sub-tasks in parallel
            results = parallel_execute([
                (self.run_specialist, sub_task)
                for sub_task in batch
            ])
            for sub_task, result in zip(batch, results):
                state.specialist_results[sub_task.id] = result

        # Phase 3: Aggregate
        state.aggregated_output = self.aggregate(state)

        # Phase 4: Quality gate
        state.quality_check = self.quality_gate(state)
        if not state.quality_check["passed"]:
            return self.handle_quality_failure(state)

        state.status = "complete"
        return state

    def plan(self, task: str) -> list[SubTask]:
        """LLM-powered: decompose task into sub-tasks."""
        prompt = f"""Decompose this task into sub-tasks for specialist agents.

Task: {task}

Available specialists: {list(self.specialists.keys())}

Output a JSON array of sub-tasks, each with:
- id: unique identifier
- description: what the specialist should do
- assigned_to: specialist name
- dependencies: list of sub-task IDs this depends on
- context_needs: what data this specialist needs
"""
        return parse_plan(self.llm.complete(prompt))

    def run_specialist(self, sub_task: SubTask) -> AgentResult:
        """Run a single specialist agent with scoped context."""
        config = self.specialists[sub_task.assigned_to]
        context = self.build_specialist_context(sub_task)

        prompt = f"""{config.system_prompt}

Your task: {sub_task.description}

Context:
{context}

Expected output format: {sub_task.expected_output}
"""
        output = self.llm.complete(prompt, tools=config.tools)
        return AgentResult(
            agent_id=sub_task.assigned_to,
            sub_task_id=sub_task.id,
            output=output,
            confidence=0.0,  # assessed by quality gate
            metadata={},
        )

    def aggregate(self, state: MultiAgentState) -> str:
        """LLM-powered: merge specialist outputs into coherent result."""
        results_summary = "\n\n".join([
            f"--- {r.agent_id} (task: {r.sub_task_id}) ---\n{r.output}"
            for r in state.specialist_results.values()
        ])
        prompt = f"""Synthesize these specialist outputs into a single coherent response.

Original task: {state.task}

Specialist outputs:
{results_summary}

Merge these into a unified, well-structured response. Resolve any contradictions
by favoring the specialist with the most relevant expertise.
"""
        return self.llm.complete(prompt)

    def quality_gate(self, state: MultiAgentState) -> dict:
        """Code-based: validate aggregated output."""
        checks = {
            "all_subtasks_complete": all(
                r.output for r in state.specialist_results.values()
            ),
            "output_not_empty": len(state.aggregated_output) > 0,
            "no_contradictions": self.check_consistency(state),
        }
        return {
            "passed": all(checks.values()),
            "checks": checks,
        }
```

## Error Handling

### Specialist Failure Modes

| Failure | Detection | Response |
|---------|-----------|----------|
| **Timeout** | Deadline exceeded | Retry once, then continue without that specialist's output |
| **Low quality output** | Quality gate rejects | Re-run specialist with more specific instructions |
| **Contradiction** | Aggregator detects conflicting specialist outputs | Orchestrator resolves by re-querying with both outputs |
| **Total failure** | Exception or empty output | Log, mark sub-task as failed, proceed with partial results |

### Graceful Degradation

The multi-agent pattern should degrade gracefully. If one specialist fails:

1. Mark the sub-task as failed in state.
2. Check if downstream sub-tasks depend on the failed one.
3. If dependent sub-tasks exist, attempt to provide them with partial context.
4. If the aggregator can produce a useful result without the missing specialist, proceed.
5. Flag the output as partial and inform the user.

## Example: Research System with Parallel Agents

### Task
"Analyze the competitive landscape for AI code generation tools."

### Plan (generated by orchestrator)

```
Sub-task 1: "Market Research"
  Assigned to: researcher
  Dependencies: []
  Context: industry reports, market data

Sub-task 2: "Product Analysis"
  Assigned to: analyst
  Dependencies: []
  Context: product features, pricing, user reviews

Sub-task 3: "Technical Comparison"
  Assigned to: engineer
  Dependencies: []
  Context: architecture docs, benchmarks, API documentation

Sub-task 4: "Synthesis Report"
  Assigned to: writer
  Dependencies: [1, 2, 3]
  Context: outputs from all previous sub-tasks
```

### Execution

```
Time -->
  T0: Orchestrator creates plan
  T1: [researcher] [analyst] [engineer]  -- parallel execution
  T2: Writer receives all outputs
  T3: Orchestrator runs quality gate
  T4: Final report delivered
```

### Result

The orchestrator aggregates three specialist outputs into a structured competitive analysis that no single agent could produce as efficiently alone.

## Design Guidelines

1. **The orchestrator is the single source of truth.** Never let specialists modify shared state directly.
2. **Prefer parallel over sequential.** If sub-tasks are independent, run them simultaneously.
3. **Keep specialist prompts focused.** Each specialist should have a clear, narrow mandate. A specialist prompt should be shorter than the original task prompt.
4. **Budget context carefully.** Each specialist gets only the context it needs, not the full shared state.
5. **Log all inter-agent messages.** This is your primary debugging tool for multi-agent issues.
6. **Start with 2-3 specialists.** Add more only when you can demonstrate that the added coordination cost is justified by output quality improvement.
