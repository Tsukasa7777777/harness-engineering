# Sub-agent Delegation Pattern

A primary agent decomposes a large task into smaller, independent sub-tasks and delegates each to a sub-agent with scoped context. The primary agent aggregates results and maintains the high-level picture.

## When to Use

- The task is too large for a single context window (large codebase, extensive research).
- Sub-tasks are independent and benefit from focused context rather than shared context.
- You want to keep the primary agent's context window lean and strategic.
- Parallel execution of sub-tasks would significantly reduce total time.

## When NOT to Use

- Sub-tasks are tightly interdependent (each step needs the output of the previous one; use Sequential).
- The task requires shared reasoning across all sub-tasks (use a single agent with careful context management).
- The decomposition overhead exceeds the benefit (task is small enough for one agent).

## Architecture

```
+-----------------------------+
|       Primary Agent         |
|  (Planner + Aggregator)     |
|                             |
|  Responsibilities:          |
|  - Decompose task           |
|  - Select sub-agents        |
|  - Pass scoped context      |
|  - Aggregate results        |
|  - Maintain overall state   |
+-----------------------------+
       |    |    |    |
       v    v    v    v
    +----+ +----+ +----+ +----+
    |SA-1| |SA-2| |SA-3| |SA-4|   SA = Sub-Agent
    +----+ +----+ +----+ +----+
       |    |    |    |
       v    v    v    v
+-----------------------------+
|       Primary Agent         |
|  (Result Aggregation)       |
|                             |
|  - Merge sub-agent outputs  |
|  - Resolve conflicts        |
|  - Fill gaps                |
|  - Produce final output     |
+-----------------------------+
```

### Difference from Multi-Agent

| Aspect | Multi-Agent | Delegation |
|--------|-------------|------------|
| **Agent types** | Permanent specialist roles (researcher, analyst, writer) | Ephemeral instances of the same general agent |
| **Context** | Each specialist has unique expertise/prompts | All sub-agents share the same base prompt, differ only in scoped context |
| **Lifespan** | Specialists persist across tasks | Sub-agents are created and destroyed per task |
| **Primary use case** | Tasks requiring diverse expertise | Tasks requiring broad coverage |

## Task Decomposition Strategy

### Decomposition Principles

1. **Independence**: Sub-tasks should be completable without knowledge of other sub-tasks' results.
2. **Completeness**: The union of all sub-tasks should cover the entire original task.
3. **Bounded scope**: Each sub-task should fit within a single context window with room to spare.
4. **Clear boundaries**: It should be unambiguous which part of the work belongs to which sub-task.

### Decomposition Approaches

| Approach | Example | Best For |
|----------|---------|----------|
| **By file/module** | "Analyze files A-D", "Analyze files E-H" | Codebase exploration |
| **By topic** | "Research pricing", "Research features" | Research tasks |
| **By time period** | "Summarize Q1", "Summarize Q2" | Historical analysis |
| **By entity** | "Profile competitor A", "Profile competitor B" | Entity-focused tasks |
| **By concern** | "Check security", "Check performance" | Multi-concern analysis |

### Decomposition Template

```python
def decompose_task(task: str, constraints: dict) -> list[SubTask]:
    """Decompose a task into sub-tasks for delegation.

    The decomposition itself can be done by code (for structured tasks)
    or by LLM (for unstructured tasks).
    """
    # Code-based decomposition (preferred when structure is known)
    if constraints.get("decomposition") == "by_file":
        files = list_files(constraints["directory"])
        batches = chunk(files, constraints.get("batch_size", 10))
        return [
            SubTask(
                id=f"batch-{i}",
                description=f"Analyze these files for {task}",
                context={"files": batch},
                expected_output="Analysis findings as structured JSON",
            )
            for i, batch in enumerate(batches)
        ]

    # LLM-based decomposition (for unstructured tasks)
    prompt = f"""Decompose this task into independent sub-tasks:

Task: {task}

Rules:
- Each sub-task must be completable independently
- Each sub-task must be small enough for a single focused analysis
- Together, all sub-tasks must fully cover the original task
- Clearly define what each sub-task should output

Output as JSON array of sub-tasks.
"""
    return parse_subtasks(llm.complete(prompt))
```

## Context Passing

### Context Scoping

The primary agent must decide what context each sub-agent receives. This is a critical engineering decision: too much context wastes tokens and confuses the sub-agent; too little context causes hallucination or incomplete work.

```python
@dataclass
class SubAgentContext:
    """Context envelope passed to a sub-agent."""
    # Global context (shared across all sub-agents)
    task_summary: str        # One-paragraph summary of the overall task
    output_format: str       # Expected output format specification
    constraints: list[str]   # Global constraints that apply to all sub-tasks

    # Scoped context (unique to this sub-agent)
    sub_task: str            # Specific sub-task description
    relevant_data: dict      # Data relevant to this specific sub-task
    file_contents: list[str] # File contents (if applicable)

    # Boundaries
    scope_boundary: str      # Explicit statement of what is IN and OUT of scope
```

### Context Assembly

```python
def build_sub_agent_context(task, sub_task, global_context) -> str:
    """Assemble the prompt for a sub-agent."""
    return f"""You are a focused analysis agent. You have ONE job.

OVERALL TASK (for context only, do NOT try to complete the full task):
{task.summary}

YOUR SPECIFIC SUB-TASK:
{sub_task.description}

YOUR SCOPE:
- IN scope: {sub_task.scope_boundary}
- OUT of scope: Everything else. Do not attempt work outside your scope.

RELEVANT DATA:
{format_data(sub_task.relevant_data)}

OUTPUT FORMAT:
{task.output_format}

CONSTRAINTS:
{format_constraints(task.constraints)}

Complete your sub-task now. Focus only on your assigned scope.
"""
```

### Anti-Patterns

| Anti-Pattern | Problem | Fix |
|--------------|---------|-----|
| Passing the full original prompt | Sub-agent tries to do everything | Scope the prompt to the sub-task only |
| No scope boundary | Sub-agent wanders into other sub-tasks' territory | Explicitly state what is in and out of scope |
| Omitting the overall task summary | Sub-agent lacks strategic context | Include a brief summary for orientation |
| Passing all data | Context window bloat, confusion | Pre-filter to relevant data only |

## Result Aggregation

### Aggregation Strategy

```python
def aggregate_results(task, sub_results: list[SubAgentResult]) -> str:
    """Merge sub-agent results into a coherent output."""

    # Step 1: Validate all results
    valid_results = []
    failed_results = []
    for result in sub_results:
        if result.status == "success" and result.output:
            valid_results.append(result)
        else:
            failed_results.append(result)

    if failed_results:
        log(f"Warning: {len(failed_results)} sub-tasks failed")

    # Step 2: Check for conflicts
    conflicts = detect_conflicts(valid_results)
    if conflicts:
        log(f"Conflicts detected: {conflicts}")

    # Step 3: Merge (LLM-powered if needed)
    if needs_llm_merge(valid_results):
        return llm_merge(task, valid_results, conflicts)
    else:
        return deterministic_merge(valid_results)

def llm_merge(task, results, conflicts):
    """Use LLM to synthesize sub-agent outputs."""
    prompt = f"""Merge these sub-agent outputs into a single coherent response.

Original task: {task.summary}

Sub-agent outputs:
{format_results(results)}

{f"Conflicts to resolve: {conflicts}" if conflicts else ""}

Rules:
- Preserve all unique findings from each sub-agent
- Resolve conflicts by noting both perspectives
- Organize the merged output logically
- Remove redundancy
- Ensure the final output fully addresses the original task
"""
    return llm.complete(prompt)
```

### Merge Strategies

| Strategy | When | Implementation |
|----------|------|----------------|
| **Concatenate** | Results are independent sections | Join with headers |
| **Deduplicate** | Overlapping coverage | Extract unique findings, remove duplicates |
| **Synthesize** | Results need integration into a narrative | LLM-powered merge |
| **Rank** | Results are alternatives, not complements | Score and select best |
| **Vote** | Results are judgments on the same question | Majority wins |

## Failure Handling

### Failure Modes

| Mode | Detection | Impact | Response |
|------|-----------|--------|----------|
| **Sub-agent timeout** | Deadline exceeded | One sub-task missing | Mark as incomplete, aggregate available results |
| **Sub-agent error** | Exception thrown | One sub-task failed | Retry once, then mark as failed |
| **Low quality output** | Validation check fails | Unreliable sub-task | Retry with more specific instructions |
| **All sub-agents fail** | No successful results | Complete failure | Escalate to human, provide diagnostic info |
| **Partial coverage** | Aggregation detects gaps | Incomplete output | Launch additional sub-agents for gaps |

### Retry Strategy

```python
def run_sub_agent_with_retry(sub_task, max_retries=2):
    """Run a sub-agent with retry logic."""
    for attempt in range(max_retries + 1):
        try:
            result = run_sub_agent(sub_task)

            # Validate output quality
            if validate_sub_output(result):
                return result

            # Quality issue -- retry with feedback
            sub_task.context["previous_attempt"] = result.output
            sub_task.context["quality_issues"] = result.quality_issues
            log(f"Sub-agent quality issue, retrying ({attempt + 1}/{max_retries})")

        except TimeoutError:
            log(f"Sub-agent timeout, retrying ({attempt + 1}/{max_retries})")
        except Exception as e:
            log(f"Sub-agent error: {e}, retrying ({attempt + 1}/{max_retries})")

    return SubAgentResult(status="failed", output=None, error="Max retries exceeded")
```

### Graceful Degradation

```python
def aggregate_with_degradation(task, results):
    """Aggregate results, handling missing sub-tasks."""
    successful = [r for r in results if r.status == "success"]
    failed = [r for r in results if r.status == "failed"]

    if not successful:
        return DegradedResult(
            status="total_failure",
            message="All sub-agents failed. Manual completion required.",
            diagnostics=[r.error for r in failed],
        )

    coverage = len(successful) / len(results)
    if coverage < 0.5:
        return DegradedResult(
            status="partial",
            output=aggregate(successful),
            message=f"Only {coverage:.0%} of sub-tasks completed. Output is incomplete.",
            missing=[r.sub_task_id for r in failed],
        )

    return FullResult(
        status="complete" if not failed else "mostly_complete",
        output=aggregate(successful),
        warnings=[f"Sub-task {r.sub_task_id} failed" for r in failed],
    )
```

## Example: Large Codebase Exploration

### Task
"Find all usages of deprecated API calls in this 500-file codebase and suggest replacements."

### Decomposition

```python
# Code-based decomposition: split by directory
directories = ["src/auth/", "src/api/", "src/models/", "src/utils/", "src/views/"]

sub_tasks = [
    SubTask(
        id=f"scan-{dir_name}",
        description=f"Find deprecated API calls in {dir_name} and suggest replacements",
        context={
            "files": list_files(dir_name),
            "deprecated_apis": DEPRECATED_API_LIST,
            "replacement_map": REPLACEMENT_MAP,
        },
        scope_boundary=f"Only files in {dir_name}. Do not analyze other directories.",
    )
    for dir_name in directories
]
```

### Execution

```
Primary Agent: "Scanning 500 files across 5 directories for deprecated APIs."

Sub-agent 1 (src/auth/):    Found 3 deprecated calls    [2.1s]
Sub-agent 2 (src/api/):     Found 12 deprecated calls   [3.4s]
Sub-agent 3 (src/models/):  Found 0 deprecated calls    [1.8s]
Sub-agent 4 (src/utils/):   Found 7 deprecated calls    [2.6s]
Sub-agent 5 (src/views/):   Found 5 deprecated calls    [2.9s]

Total time: 3.4s (parallel) vs ~12.8s (sequential)
```

### Aggregation

Primary agent merges all findings into a single report:
- Sorted by severity
- Grouped by deprecated API
- Each finding includes file, line, current code, and suggested replacement
- Summary statistics at the top

## Design Guidelines

1. **Prefer code-based decomposition.** If you can split by file, directory, entity, or other structural boundary, do it in code. LLM-based decomposition is a last resort.
2. **Scope aggressively.** Sub-agents should receive the minimum context needed. The primary agent holds the big picture.
3. **Run in parallel by default.** If sub-tasks are independent (they should be), run them concurrently.
4. **Plan for partial failure.** Your aggregation logic should always handle missing sub-task results.
5. **Keep the primary agent lean.** It should plan, delegate, and aggregate -- not do detailed work itself. All detailed work goes to sub-agents.
6. **Set timeouts on sub-agents.** A stuck sub-agent should not block the entire workflow.
