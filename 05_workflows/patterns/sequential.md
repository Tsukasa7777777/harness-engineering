# Sequential Workflow Pattern

A linear pipeline where each step completes before the next begins. The harness owns the step order; the LLM provides intelligence at specific steps.

## When to Use

- The task has a well-understood sequence of operations.
- Steps have clear dependencies (step N requires the output of step N-1).
- The workflow is deterministic in structure, even if individual steps involve LLM reasoning.
- You are building a first version and want the simplest reliable pattern.

## When NOT to Use

- Steps are independent and could run in parallel (use Multi-Agent or Delegation).
- Output quality requires iterative refinement (use Iterative).
- The step sequence itself needs to be determined dynamically (use a router or planner pattern).

## Architecture

```
 START
   |
   v
+------------------+
| Step 1: Gather   |  <-- Deterministic (tools, API calls)
| Context          |
+------------------+
   |
   v
+------------------+
| Step 2: Analyze  |  <-- LLM-powered (reasoning, judgment)
|                  |
+------------------+
   |
   v
+------------------+
| Step 3: Generate |  <-- LLM-powered (content creation)
| Output           |
+------------------+
   |
   v
+------------------+
| Step 4: Validate |  <-- Deterministic (schema check, tests)
|                  |
+------------------+
   |
   v
+------------------+
| Step 5: Format   |  <-- Deterministic (templating)
| & Deliver        |
+------------------+
   |
   v
  END
```

Key: Each step has a single responsibility. Deterministic steps use tools/code. LLM steps use model calls. The harness transitions between steps.

## Implementation Template

```python
from dataclasses import dataclass, field
from typing import Any
from enum import Enum

class StepStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    SKIPPED = "skipped"

@dataclass
class WorkflowState:
    """Explicit working state owned by the harness."""
    task: str
    context: dict = field(default_factory=dict)
    analysis: str = ""
    output: str = ""
    validation_result: dict = field(default_factory=dict)
    final_output: str = ""
    current_step: int = 0
    step_history: list = field(default_factory=list)
    error: str | None = None

class SequentialWorkflow:
    def __init__(self, llm_client, tools):
        self.llm = llm_client
        self.tools = tools
        self.steps = [
            ("gather_context", self.gather_context),
            ("analyze", self.analyze),
            ("generate_output", self.generate_output),
            ("validate", self.validate),
            ("format_and_deliver", self.format_and_deliver),
        ]

    def run(self, task: str) -> WorkflowState:
        state = WorkflowState(task=task)

        for i, (step_name, step_fn) in enumerate(self.steps):
            state.current_step = i
            log(f"Starting step {i}: {step_name}")

            try:
                state = step_fn(state)
                state.step_history.append({
                    "step": step_name,
                    "status": StepStatus.COMPLETED.value,
                })
                log(f"Completed step {i}: {step_name}")

            except SkipStep:
                state.step_history.append({
                    "step": step_name,
                    "status": StepStatus.SKIPPED.value,
                })
                log(f"Skipped step {i}: {step_name}")
                continue

            except Exception as e:
                state.error = str(e)
                state.step_history.append({
                    "step": step_name,
                    "status": StepStatus.FAILED.value,
                    "error": str(e),
                })
                log(f"Failed step {i}: {step_name} -- {e}")
                return self.handle_failure(state, step_name, e)

        return state

    # --- Step implementations ---

    def gather_context(self, state: WorkflowState) -> WorkflowState:
        """Deterministic: fetch data using tools."""
        state.context = self.tools.fetch_relevant_data(state.task)
        return state

    def analyze(self, state: WorkflowState) -> WorkflowState:
        """LLM-powered: reason about the gathered context."""
        prompt = build_analysis_prompt(state.task, state.context)
        state.analysis = self.llm.complete(prompt)
        return state

    def generate_output(self, state: WorkflowState) -> WorkflowState:
        """LLM-powered: produce the main output."""
        prompt = build_generation_prompt(state.task, state.analysis)
        state.output = self.llm.complete(prompt)
        return state

    def validate(self, state: WorkflowState) -> WorkflowState:
        """Deterministic: check output against criteria."""
        state.validation_result = self.tools.validate(state.output)
        if not state.validation_result["passed"]:
            raise ValidationError(state.validation_result["errors"])
        return state

    def format_and_deliver(self, state: WorkflowState) -> WorkflowState:
        """Deterministic: format and deliver the output."""
        state.final_output = self.tools.format(state.output)
        self.tools.deliver(state.final_output)
        return state

    def handle_failure(self, state, step_name, error):
        """Centralized failure handling."""
        log(f"Workflow failed at {step_name}: {error}")
        # Options: retry, fallback, escalate to human
        return state
```

## Error Handling Strategy

Sequential workflows have a straightforward error model because the linear structure makes it clear where failures occur.

### Per-Step Error Handling

| Error Type | Strategy | Example |
|------------|----------|---------|
| **Transient** (network, rate limit) | Retry with exponential backoff (max 3 attempts) | API call timeout |
| **Validation** (output doesn't meet criteria) | Retry the step with error context appended to prompt | LLM output missing required fields |
| **Fatal** (unrecoverable) | Log, save state, escalate to human | Authentication failure |
| **Skip** (step not applicable) | Skip and continue to next step | Optional enrichment step |

### Retry Pattern

```python
def retry_step(step_fn, state, max_retries=3):
    for attempt in range(max_retries):
        try:
            return step_fn(state)
        except TransientError as e:
            if attempt == max_retries - 1:
                raise
            wait = 2 ** attempt  # exponential backoff
            sleep(wait)
    raise MaxRetriesExceeded()
```

### State Checkpoint

After each successful step, serialize the state. On failure, you can resume from the last checkpoint instead of restarting.

```python
def run_with_checkpoints(self, task, resume_from=None):
    state = resume_from or WorkflowState(task=task)
    start = state.current_step

    for i, (name, fn) in enumerate(self.steps[start:], start=start):
        state = fn(state)
        save_checkpoint(state)  # persist after each step

    return state
```

## Example Use Cases

### 1. Daily Report Generation

```
Step 1: Fetch calendar events, completed tasks, git commits  (tools)
Step 2: Analyze what was accomplished and what was missed     (LLM)
Step 3: Generate a structured daily report                    (LLM)
Step 4: Validate report has all required sections             (code)
Step 5: Post to Notion and send via Slack                     (tools)
```

### 2. Customer Proposal Creation

```
Step 1: Pull customer info, project requirements, pricing     (tools)
Step 2: Analyze requirements and identify key selling points   (LLM)
Step 3: Generate proposal document                             (LLM)
Step 4: Validate pricing calculations and formatting           (code)
Step 5: Export as PDF and email to customer                    (tools)
```

### 3. Code Review Pipeline

```
Step 1: Fetch PR diff from GitHub                              (tools)
Step 2: Analyze code changes for issues, style, patterns       (LLM)
Step 3: Generate review comments with specific line references (LLM)
Step 4: Validate comments reference valid lines and files      (code)
Step 5: Post review comments to GitHub PR                      (tools)
```

## Design Guidelines

1. **Keep steps coarse-grained.** 3-7 steps is typical. If you have 15 steps, some can likely be merged.
2. **Alternate deterministic and LLM steps.** This creates natural validation boundaries.
3. **Make each step independently testable.** You should be able to unit test a step by providing a mock state.
4. **Log the state transition at each boundary.** This is your primary debugging tool.
5. **Default to this pattern.** Start sequential. Evolve to more complex patterns only when you hit a concrete limitation.
