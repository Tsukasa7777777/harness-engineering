# Runbook: New Project Setup

Step-by-step procedure for bootstrapping harness engineering on a new project. Follow these steps in order. Each step includes verification criteria before proceeding.

---

## Prerequisites

Before starting, ensure you have:

- [ ] Repository access (read/write)
- [ ] LLM API access configured and tested
- [ ] MCP server connections verified (if applicable)
- [ ] Understanding of the project's purpose and requirements

---

## Step 1: Initialize CLAUDE.md

**Time estimate**: 15-30 minutes

### What

Create the root `CLAUDE.md` file that serves as the agent's operating manual. This file defines the agent's role, available tools, constraints, and behavioral expectations.

### How

```markdown
# Project: [Name]

## Agent Role
[One paragraph: what the agent does for this project]

## Project Context
- Repository: [URL]
- Language/Framework: [tech stack]
- Team: [who the agent serves]

## Directory Structure
| Directory | Purpose |
|-----------|---------|
| ... | ... |

## Available Tools
| Tool | Purpose | Status |
|------|---------|--------|
| ... | ... | ... |

## Constraints
- [Hard constraints the agent must never violate]
- [Quality standards]
- [Security boundaries]

## Conventions
- [Naming conventions]
- [Commit message format]
- [File organization rules]
```

### Key Decisions

| Decision | Options | Recommendation |
|----------|---------|----------------|
| Scope of CLAUDE.md | Minimal vs. comprehensive | Start minimal, grow as you discover needs |
| Where to store | Root vs. `.claude/` | Root for visibility; `.claude/` if team prefers hidden config |
| Version control | Committed vs. gitignored | Always commit. This is code, not config. |

### Verification

- [ ] CLAUDE.md exists at the root of the repository
- [ ] It contains at minimum: role, project context, directory structure, and constraints
- [ ] A fresh agent session can read CLAUDE.md and understand what it should do
- [ ] No secrets, tokens, or credentials are included in CLAUDE.md

---

## Step 2: Set Up Guardrails

**Time estimate**: 30-60 minutes

### What

Define the safety boundaries that prevent the agent from causing harm. Guardrails are the non-negotiable rules that override all other instructions.

### How

Create a guardrails document (in CLAUDE.md or a separate constraints file):

```markdown
## Guardrails (Non-Negotiable)

### Prohibited Actions
- Never commit secrets, tokens, or credentials
- Never delete production data
- Never push directly to main/master
- Never modify CI/CD pipeline configuration without human approval
- Never send external communications (email, Slack) without human approval

### Required Actions
- Always run tests before marking work complete
- Always create a branch for changes (never commit to main)
- Always log actions to the daily log
- Always ask for clarification when requirements are ambiguous

### Resource Limits
- Max LLM calls per task: 50
- Max file modifications per task: 20
- Max execution time per task: 30 minutes
- Max cost per task: $5
```

### Guardrail Categories

| Category | Examples | Enforcement |
|----------|----------|-------------|
| **Destructive actions** | Delete, overwrite, push force | Block entirely or require human approval |
| **External communication** | Email, Slack, API calls to external services | Require human approval |
| **Resource consumption** | LLM calls, API calls, compute time | Hard limits in the harness |
| **Data handling** | Customer data, credentials, PII | Never include in outputs or logs |
| **Scope** | Files to touch, repos to modify, systems to access | Whitelist approach |

### Verification

- [ ] Guardrails document exists and is referenced from CLAUDE.md
- [ ] Each prohibited action has a clear enforcement mechanism (not just a "don't")
- [ ] Resource limits are defined with concrete numbers
- [ ] The agent can explain the guardrails when asked "What are you not allowed to do?"

---

## Step 3: Define Evaluation Criteria

**Time estimate**: 30-60 minutes

### What

Define how you will measure whether the agent is producing good output. Without evaluation criteria, you cannot improve the system.

### How

Create an evaluation framework with three layers:

#### Layer 1: Automated Checks (run on every output)

```yaml
automated_checks:
  - name: "syntax_valid"
    description: "Output is syntactically valid (code compiles, JSON parses)"
    type: deterministic
    implementation: "language-specific linter/parser"

  - name: "tests_pass"
    description: "All existing tests continue to pass"
    type: deterministic
    implementation: "test suite runner"

  - name: "no_secrets"
    description: "Output does not contain secrets or credentials"
    type: deterministic
    implementation: "secret scanner (gitleaks, trufflehog)"

  - name: "within_scope"
    description: "Changes are limited to files within the task scope"
    type: deterministic
    implementation: "diff analysis against allowed file list"
```

#### Layer 2: Quality Metrics (tracked over time)

```yaml
quality_metrics:
  - name: "first_pass_success_rate"
    description: "Percentage of tasks completed correctly on first attempt"
    target: ">80%"
    measurement: "manual review of completed tasks"

  - name: "iteration_count"
    description: "Average number of iterations before convergence"
    target: "<3"
    measurement: "logged iteration counts"

  - name: "human_override_rate"
    description: "Percentage of tasks requiring human intervention"
    target: "<20%"
    measurement: "logged override events"
```

#### Layer 3: Periodic Human Review (weekly/monthly)

```yaml
human_review:
  frequency: "weekly"
  sample_size: "5 tasks or 10%, whichever is larger"
  criteria:
    - "Output quality matches senior engineer standard"
    - "Agent followed guardrails"
    - "Agent asked for clarification when appropriate"
    - "No unnecessary work or scope creep"
```

### Verification

- [ ] Automated checks are implemented and runnable
- [ ] Quality metrics are defined with concrete targets
- [ ] Human review process is documented with frequency and criteria
- [ ] There is a mechanism to feed review results back into prompt improvements

---

## Step 4: Configure Tools

**Time estimate**: 30-120 minutes (depends on tool count)

### What

Set up and verify all MCP servers, API connections, and tool integrations the agent needs.

### How

#### 4.1 Inventory Required Tools

List every external system the agent needs to interact with:

```markdown
| Tool | Required For | Priority | Status |
|------|-------------|----------|--------|
| GitHub | Code management, PRs, issues | Must-have | Not configured |
| Notion | Documentation, knowledge base | Must-have | Not configured |
| Slack | Team communication | Nice-to-have | Not configured |
```

#### 4.2 Configure Each Tool

For each tool:

1. Install the MCP server or API client
2. Set up authentication (store credentials securely)
3. Test the connection with a read-only operation
4. Test a write operation (in a safe/sandbox context)
5. Document the configuration

#### 4.3 Tool Configuration Checklist

For each tool, verify:

```markdown
### [Tool Name]
- [ ] MCP server installed and running
- [ ] Authentication configured (credentials in secure storage, not in repo)
- [ ] Read operation tested successfully
- [ ] Write operation tested successfully
- [ ] Error handling verified (what happens when the tool is unavailable?)
- [ ] Rate limits documented and respected
- [ ] Tool is listed in CLAUDE.md with status
```

#### 4.4 Tool Fallback Strategy

```markdown
| Tool | Fallback if Unavailable | Impact |
|------|------------------------|--------|
| GitHub | CLI commands via bash | Reduced convenience, same capability |
| Notion | Direct API calls | Reduced convenience, same capability |
| Slack | Skip notification, log instead | Human misses update, task still completes |
```

### Verification

- [ ] All must-have tools are configured and tested
- [ ] Each tool has a verified read and write operation
- [ ] Fallback strategies are defined for tool unavailability
- [ ] CLAUDE.md tool table is updated with current status

---

## Step 5: First Test Run

**Time estimate**: 30-60 minutes

### What

Execute a representative task end-to-end to verify the entire harness works together. Choose a task that exercises the most common workflow.

### How

#### 5.1 Select a Test Task

Choose a task that:
- Is representative of typical work
- Is low-risk (easy to undo if something goes wrong)
- Exercises multiple tools
- Has clear success criteria

Good test tasks:
- Generate a daily summary from calendar + tasks
- Create a PR from a simple code change
- Produce a meeting notes document from a transcript
- Triage and categorize a batch of issues

#### 5.2 Execute with Full Logging

Run the task with maximum observability:

```bash
# Enable verbose logging
export AGENT_LOG_LEVEL=debug

# Run the task
# (your specific invocation command)

# Capture the output
# Logs should capture: every LLM call, every tool call, every state transition
```

#### 5.3 Review Checklist

After the test run, verify:

```markdown
### Execution Review
- [ ] Task completed without errors
- [ ] All expected tools were used
- [ ] Output matches expected format
- [ ] Output meets quality criteria
- [ ] Guardrails were respected (no prohibited actions taken)
- [ ] Execution time is acceptable
- [ ] Token usage is within budget

### Observability Review
- [ ] Logs capture every step
- [ ] State transitions are visible in logs
- [ ] Tool calls and responses are logged
- [ ] Errors (if any) are clearly described in logs
- [ ] You could debug a failure from the logs alone

### Edge Case Review
- [ ] What happens if the primary LLM is unavailable? (test with timeout)
- [ ] What happens if a tool returns an error? (test with invalid input)
- [ ] What happens if the output fails validation? (test with intentionally bad prompt)
```

#### 5.4 Document Findings

Create a test results document:

```markdown
## First Test Run Results

Date: [date]
Task: [description]
Duration: [time]
Token usage: [count]
Outcome: [pass/fail]

### What Worked
- [item]

### What Failed
- [item]: [root cause], [fix]

### Action Items
- [ ] [fix/improvement to make before next run]
```

### Verification

- [ ] Test task completed successfully (or failures are documented with fixes)
- [ ] Observability is sufficient to debug issues
- [ ] Edge cases have been explored
- [ ] Action items from the test run have been captured

---

## Post-Setup Checklist

After completing all 5 steps, run this final checklist:

```markdown
## Project Readiness

### Configuration
- [ ] CLAUDE.md is complete and committed
- [ ] Guardrails are defined and enforceable
- [ ] Evaluation criteria are defined with automated checks
- [ ] All required tools are configured and tested
- [ ] First test run completed successfully

### Documentation
- [ ] Setup decisions are documented (why, not just what)
- [ ] Tool configurations are documented
- [ ] Guardrails are documented and accessible to the agent
- [ ] Test results are documented

### Operational Readiness
- [ ] Logging is configured and capturing useful data
- [ ] Error handling is in place for common failure modes
- [ ] Human escalation path is defined
- [ ] First real task is identified and ready to execute
```

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| CLAUDE.md too long | Agent ignores parts of it | Keep under 500 lines. Use references for details. |
| Guardrails too vague | Agent finds loopholes | Use specific, testable statements. |
| No evaluation criteria | Cannot tell if agent is improving | Define metrics before running tasks. |
| Tools configured but not tested | Failures during real tasks | Always test read + write for every tool. |
| Skipping the test run | First real task hits preventable issues | The test run is not optional. |
