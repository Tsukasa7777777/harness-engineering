# Iterative Refinement Pattern

A feedback loop where agent output is evaluated against quality criteria, and the agent refines its output until the criteria are met or an iteration limit is reached.

## When to Use

- Output quality is verifiable by automated checks (tests pass, schema validates, linter clean).
- First-pass output is often "close but not right" and benefits from self-correction.
- The cost of iteration is lower than the cost of delivering bad output.
- You have clear, programmatic success criteria (not just "looks good").

## When NOT to Use

- There are no automated quality checks (human judgment is the only evaluator).
- First-pass output is usually correct (iteration adds cost without benefit).
- The task is latency-sensitive and cannot tolerate multiple LLM round-trips.
- Quality criteria are subjective or ambiguous (the agent will oscillate rather than converge).

## Architecture

```
              +------------------+
              |  Initial Prompt  |
              |  + Context       |
              +------------------+
                       |
                       v
              +------------------+
         +--->|  LLM Generation  |
         |    +------------------+
         |             |
         |             v
         |    +------------------+
         |    |   Quality Gate   |  <-- Deterministic checks
         |    | (tests, linter,  |
         |    |  schema, eval)   |
         |    +------------------+
         |          |         |
         |        FAIL       PASS
         |          |         |
         |          v         v
         |    +-----------+  +--------+
         |    | Build     |  | Output |
         +----| Feedback  |  | Final  |
              | Prompt    |  | Result |
              +-----------+  +--------+

         Loop invariant: iteration_count < MAX_ITERATIONS
```

## Feedback Loop Design

### Core Loop

```python
def iterative_refinement(task, context, max_iterations=5):
    state = IterativeState(
        task=task,
        context=context,
        iteration=0,
        output=None,
        feedback_history=[],
    )

    while state.iteration < max_iterations:
        state.iteration += 1
        log(f"Iteration {state.iteration}/{max_iterations}")

        # Generate (or refine)
        if state.iteration == 1:
            state.output = generate_initial(state)
        else:
            state.output = refine(state)

        # Evaluate
        evaluation = quality_gate(state.output)

        if evaluation.passed:
            log(f"Quality gate passed on iteration {state.iteration}")
            state.status = "converged"
            return state

        # Record feedback
        state.feedback_history.append({
            "iteration": state.iteration,
            "output_summary": summarize(state.output),
            "evaluation": evaluation,
        })
        log(f"Quality gate failed: {evaluation.issues}")

    # Exhausted iterations
    state.status = "max_iterations_reached"
    log(f"Did not converge after {max_iterations} iterations")
    return state
```

### Feedback Prompt Construction

The feedback prompt is the critical differentiator between productive iteration and aimless looping. It must be specific, actionable, and cumulative.

```python
def build_feedback_prompt(state):
    """Construct a prompt that gives the LLM specific, actionable feedback."""
    return f"""Your previous output did not pass quality checks.

Original task: {state.task}

Your previous output:
{state.output}

Issues found:
{format_issues(state.feedback_history[-1]["evaluation"].issues)}

Previous attempts and their issues:
{format_history(state.feedback_history)}

Instructions:
1. Fix ONLY the issues listed above.
2. Do not change parts that were already correct.
3. Explain what you changed and why.

Produce the corrected output:
"""
```

### Key Design Principles

1. **Feedback must be specific.** "Output is wrong" is useless. "Line 15: variable `x` is undefined" is actionable.
2. **Include the full history.** The LLM must see all previous attempts and their failures to avoid repeating mistakes.
3. **Separate what is correct from what is wrong.** Prevent the LLM from "fixing" things that were already right.
4. **Limit context growth.** Summarize old iterations rather than including full text of every attempt.

## Convergence Criteria

### Types of Quality Gates

| Gate Type | Example | Determinism |
|-----------|---------|-------------|
| **Syntax** | Code parses, JSON validates, Markdown renders | Fully deterministic |
| **Tests** | Unit tests pass, integration tests pass | Fully deterministic |
| **Schema** | Output matches JSON Schema, has required fields | Fully deterministic |
| **Lint** | No lint errors, style guide compliance | Fully deterministic |
| **Semantic** | LLM evaluator scores output > threshold | Non-deterministic |
| **Composite** | All of the above must pass | Mixed |

**Strongly prefer deterministic gates.** Non-deterministic gates (LLM-as-judge) should be used only as a last resort because they introduce instability in the convergence loop.

### Convergence Detection

```python
@dataclass
class Evaluation:
    passed: bool
    issues: list[str]
    score: float  # 0.0-1.0
    issue_count: int

def detect_convergence(history: list[Evaluation]) -> str:
    """Detect convergence patterns to avoid wasted iterations."""
    if len(history) < 2:
        return "continue"

    current = history[-1]
    previous = history[-2]

    # Converged: no issues
    if current.passed:
        return "converged"

    # Improving: fewer issues than last time
    if current.issue_count < previous.issue_count:
        return "continue"

    # Stalled: same number of issues
    if current.issue_count == previous.issue_count:
        return "stalled"

    # Regressing: more issues than last time
    if current.issue_count > previous.issue_count:
        return "regressing"

    return "continue"
```

### Handling Non-Convergence

| Pattern | Response |
|---------|----------|
| **Improving** | Continue iterating |
| **Stalled** | Increase prompt specificity, add examples, or change strategy |
| **Regressing** | Revert to best previous attempt, try a different approach |
| **Oscillating** (alternating between two states) | Fix one issue set at a time, not all at once |

## Max Iteration Limits

### Setting the Limit

| Task Complexity | Recommended Max | Rationale |
|-----------------|-----------------|-----------|
| Simple formatting fixes | 2-3 | Should converge quickly or not at all |
| Code generation with tests | 3-5 | Most issues resolve in 2-3 iterations |
| Complex document generation | 3-5 | Beyond 5, diminishing returns |
| Multi-constraint optimization | 5-8 | More constraints need more passes |

### Exhaustion Strategy

When the iteration limit is reached without convergence:

```python
def handle_exhaustion(state):
    """Strategy when max iterations reached."""
    # Option 1: Return best attempt
    best = select_best_attempt(state.feedback_history)

    # Option 2: Return with warnings
    return {
        "output": best.output,
        "converged": False,
        "remaining_issues": best.evaluation.issues,
        "iterations_used": state.iteration,
        "recommendation": "Manual review required for remaining issues",
    }
```

## Quality Gates

### Gate Implementation

```python
class QualityGate:
    def __init__(self):
        self.checks = []

    def add_check(self, name, check_fn, severity="error"):
        """Register a quality check.
        severity: "error" (blocks), "warning" (logged but passes)
        """
        self.checks.append((name, check_fn, severity))

    def evaluate(self, output) -> Evaluation:
        issues = []
        warnings = []

        for name, check_fn, severity in self.checks:
            try:
                result = check_fn(output)
                if not result.passed:
                    if severity == "error":
                        issues.append(f"[{name}] {result.message}")
                    else:
                        warnings.append(f"[{name}] {result.message}")
            except Exception as e:
                issues.append(f"[{name}] Check failed with exception: {e}")

        return Evaluation(
            passed=len(issues) == 0,
            issues=issues + warnings,
            score=1.0 - (len(issues) / max(len(self.checks), 1)),
            issue_count=len(issues),
        )

# Usage
gate = QualityGate()
gate.add_check("syntax", check_python_syntax)
gate.add_check("tests", run_unit_tests)
gate.add_check("lint", run_linter, severity="warning")
gate.add_check("type_check", run_mypy)
```

### Gate Ordering

Run cheap, fast checks first. Fail early.

```
1. Syntax check       (microseconds, catches gross errors)
2. Schema validation   (milliseconds, catches structural issues)
3. Lint check          (seconds, catches style issues)
4. Unit tests          (seconds, catches logic errors)
5. Integration tests   (minutes, catches system-level issues)
```

## Example: Code Generation with Test-Fix Loop

### Task
Generate a Python function that parses CSV data with specific requirements.

### Iteration Flow

```
Iteration 1:
  Input:  "Write a function parse_csv(data: str) -> list[dict] that..."
  Output: Initial implementation
  Gate:   syntax OK, 2/5 tests fail
  Action: Continue with test failure details

Iteration 2:
  Input:  Previous output + "Tests 3,4 failed: edge case with quoted commas"
  Output: Fixed implementation
  Gate:   syntax OK, 4/5 tests pass
  Action: Continue with remaining test failure

Iteration 3:
  Input:  Previous output + "Test 5 failed: empty input should return []"
  Output: Fixed implementation
  Gate:   All tests pass, lint clean
  Action: CONVERGED -- return output
```

### Implementation

```python
def code_gen_with_tests(task, test_file, max_iterations=5):
    state = IterativeState(task=task)

    gate = QualityGate()
    gate.add_check("syntax", lambda code: check_syntax(code))
    gate.add_check("tests", lambda code: run_tests(code, test_file))
    gate.add_check("lint", lambda code: run_lint(code), severity="warning")

    for i in range(max_iterations):
        if i == 0:
            state.output = llm.complete(
                f"Write Python code for this task:\n{task}"
            )
        else:
            last_feedback = state.feedback_history[-1]
            state.output = llm.complete(build_feedback_prompt(state))

        evaluation = gate.evaluate(state.output)

        if evaluation.passed:
            return {"code": state.output, "iterations": i + 1, "converged": True}

        state.feedback_history.append({
            "iteration": i + 1,
            "evaluation": evaluation,
        })

    return {
        "code": state.output,
        "iterations": max_iterations,
        "converged": False,
        "remaining_issues": state.feedback_history[-1]["evaluation"].issues,
    }
```

## Design Guidelines

1. **Invest in the quality gate, not the prompt.** A precise quality gate matters more than a perfect initial prompt because the loop will self-correct.
2. **Make feedback machine-readable.** Test output, linter messages, and schema errors are better feedback than "it looks wrong."
3. **Summarize history to control context growth.** After 3 iterations, older attempts should be summarized, not included verbatim.
4. **Set aggressive iteration limits.** If it does not converge in 5 iterations, the problem is usually the prompt or the quality criteria, not the number of attempts.
5. **Track convergence metrics.** Plot issue counts over iterations. A healthy loop shows monotonic decrease. Oscillation or increase signals a design problem.
6. **Always have an exit.** The max iteration limit is non-negotiable. Infinite loops are the most common failure mode of iterative agents.
