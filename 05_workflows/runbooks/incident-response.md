# Runbook: Incident Response

Procedure for when an agent produces bad output, behaves unexpectedly, or causes unintended side effects. Follow these phases in order: Detect, Contain, Diagnose, Fix, Prevent Recurrence.

---

## Severity Classification

Before starting, classify the incident:

| Severity | Definition | Response Time | Examples |
|----------|-----------|---------------|----------|
| **P0 - Critical** | Agent caused damage to production systems or external-facing resources | Immediate | Deleted production data, sent incorrect external communications, committed secrets |
| **P1 - High** | Agent produced incorrect output that was or could be consumed downstream | Within 1 hour | Wrong financial calculations, incorrect customer-facing content, broken CI/CD |
| **P2 - Medium** | Agent behaved unexpectedly but damage is contained | Within 1 day | Infinite loop, excessive API calls, wrong files modified |
| **P3 - Low** | Agent produced suboptimal output but no harm done | Within 1 week | Poor quality writing, inefficient code, unnecessary steps |

---

## Phase 1: Detect

**Goal**: Identify that something went wrong and gather initial signal.

### Detection Sources

| Source | What It Catches | Latency |
|--------|----------------|---------|
| **Automated checks** (tests, linters, schema validation) | Syntax errors, test failures, format violations | Immediate |
| **Resource monitors** (token usage, API call count, execution time) | Runaway loops, excessive costs | Seconds to minutes |
| **Output review** (human or LLM-as-judge) | Quality issues, factual errors, inappropriate content | Minutes to hours |
| **Downstream systems** (CI/CD, deployment, customer reports) | Broken builds, runtime errors, user complaints | Hours to days |
| **Anomaly detection** (statistical deviation from normal behavior) | Unusual patterns in output length, tool usage, error rates | Varies |

### Detection Checklist

- [ ] What is the observed behavior? (Be specific: "Agent output X instead of expected Y")
- [ ] When did it start? (Timestamp, commit, or session ID)
- [ ] What is the blast radius? (What systems, users, or data are affected?)
- [ ] Is the agent still running? (Is the bad behavior ongoing?)
- [ ] What severity level? (Use the classification table above)

### Template: Incident Report (Initial)

```markdown
## Incident: [Brief Title]

**Detected**: [timestamp]
**Severity**: [P0/P1/P2/P3]
**Status**: Detected

### Observation
[What was observed: exact behavior, output, or side effect]

### Expected Behavior
[What should have happened instead]

### Blast Radius
[What is affected: systems, data, users]

### Agent Session
- Session ID: [if available]
- Task: [what the agent was doing]
- Last known good output: [timestamp or description]
```

---

## Phase 2: Contain

**Goal**: Stop the damage from spreading. Act first, investigate later.

### Containment Actions by Severity

#### P0 - Critical

```
1. STOP the agent immediately (kill the process/session)
2. REVERT any external changes (git revert, undo API calls, retract communications)
3. NOTIFY affected parties (team, customers, stakeholders)
4. DISABLE the agent until root cause is understood
5. PRESERVE all logs and state for diagnosis
```

#### P1 - High

```
1. STOP the agent if still running
2. QUARANTINE the output (do not distribute or deploy)
3. ASSESS whether downstream systems consumed the bad output
4. NOTIFY the team
5. PRESERVE logs and state
```

#### P2 - Medium

```
1. STOP the agent if it is in a bad loop
2. SAVE the current state for diagnosis
3. NOTE the incident for later investigation
```

#### P3 - Low

```
1. NOTE the issue
2. Continue with manual correction if urgent
3. Schedule investigation for later
```

### Containment Checklist

- [ ] Agent process/session is stopped (if still running)
- [ ] External side effects are reverted or quarantined
- [ ] Affected parties are notified (if P0 or P1)
- [ ] Logs and state snapshots are preserved (do not let log rotation destroy evidence)
- [ ] The incident is documented with current status

---

## Phase 3: Diagnose

**Goal**: Find the root cause. Resist the urge to fix until you understand why.

### Diagnostic Procedure

#### 3.1 Gather Evidence

Collect these artifacts:

```markdown
### Evidence Collection
- [ ] Full agent logs for the session
- [ ] Prompt(s) sent to the LLM
- [ ] LLM response(s)
- [ ] Tool call arguments and results
- [ ] Working state at each step
- [ ] Context window contents at the point of failure
- [ ] External data that was fed to the agent
- [ ] Git diff of any changes made
- [ ] Resource usage metrics (tokens, API calls, time)
```

#### 3.2 Identify the Failure Point

Walk through the agent's execution log step by step:

```
Step 1: Was the input correct?
  |-- YES: Proceed to step 2
  |-- NO: Root cause is in input/context preparation

Step 2: Was the prompt appropriate for the task?
  |-- YES: Proceed to step 3
  |-- NO: Root cause is in prompt design

Step 3: Was the LLM response reasonable given the prompt?
  |-- YES: Proceed to step 4
  |-- NO: Root cause is LLM behavior (model issue, prompt issue, or context issue)

Step 4: Was the tool call correct (right tool, right arguments)?
  |-- YES: Proceed to step 5
  |-- NO: Root cause is in tool call generation

Step 5: Was the tool execution result correct?
  |-- YES: Proceed to step 6
  |-- NO: Root cause is in tool execution or external system

Step 6: Was the result correctly processed by the harness?
  |-- YES: Proceed to step 7
  |-- NO: Root cause is in harness logic

Step 7: Was the output correctly formatted and delivered?
  |-- YES: The failure is elsewhere (re-examine assumptions)
  |-- NO: Root cause is in output handling
```

#### 3.3 Root Cause Categories

| Category | Description | Frequency | Example |
|----------|-------------|-----------|---------|
| **Prompt defect** | Prompt is ambiguous, incomplete, or misleading | Very common | "Summarize" without specifying length or audience |
| **Context pollution** | Irrelevant or misleading context in the window | Common | Outdated data included alongside current data |
| **Tool misuse** | Agent called the wrong tool or with wrong arguments | Common | Deleting instead of archiving |
| **Missing guardrail** | Agent performed an action that should have been blocked | Moderate | No limit on iteration count |
| **LLM reasoning failure** | Model produced incorrect reasoning despite correct inputs | Moderate | Math errors, logical contradictions |
| **External system failure** | API returned error, timeout, or unexpected data | Moderate | Rate limit hit, schema change |
| **Harness bug** | Bug in the control flow, state management, or error handling code | Less common | Off-by-one in iteration counter |
| **Adversarial input** | Input contained prompt injection or adversarial content | Rare but critical | Malicious content in user-provided data |

#### 3.4 Reproduction

Attempt to reproduce the issue:

```markdown
### Reproduction Steps
1. Start with state: [serialized state or description]
2. Input: [exact input that triggered the issue]
3. Expected: [what should happen]
4. Actual: [what actually happened]
5. Reproducible: [yes/no/intermittent]
```

If the issue is intermittent, collect multiple instances and look for patterns (time of day, input characteristics, context size, etc.).

### Diagnosis Checklist

- [ ] Evidence is collected and preserved
- [ ] Failure point is identified in the execution trace
- [ ] Root cause category is determined
- [ ] Issue is reproducible (or intermittent pattern is understood)
- [ ] Root cause statement is written in plain language

---

## Phase 4: Fix

**Goal**: Implement the correction. Fix the root cause, not just the symptom.

### Fix Strategy by Root Cause

| Root Cause | Fix Approach | Testing |
|------------|-------------|---------|
| **Prompt defect** | Revise the prompt; add examples, constraints, or clarification | Re-run the exact failing input and 5 similar inputs |
| **Context pollution** | Fix the context assembly function; add filtering | Verify context contents at the failure point |
| **Tool misuse** | Add tool call validation; restrict available tools per step | Mock the tool and verify the call arguments |
| **Missing guardrail** | Add the guardrail; update the guardrails document | Attempt the prohibited action and verify it is blocked |
| **LLM reasoning failure** | Add chain-of-thought prompting; break into smaller steps; add validation | Re-run with the fix; check on 10+ diverse inputs |
| **External system failure** | Add retry logic, timeouts, and fallback handling | Simulate the failure (network timeout, error response) |
| **Harness bug** | Fix the code; add unit tests for the fixed logic | Unit test + integration test |
| **Adversarial input** | Add input sanitization; strengthen prompt instructions | Test with known adversarial inputs |

### Fix Implementation Checklist

- [ ] Fix addresses the root cause (not just the symptom)
- [ ] Fix is the minimum change needed (no unnecessary refactoring)
- [ ] Fix is tested against the original failing input
- [ ] Fix is tested against similar inputs to catch regressions
- [ ] Fix does not break existing passing tests
- [ ] Fix is code-reviewed (or self-reviewed with a clear head)
- [ ] Fix is documented (what changed and why)

### Verification Procedure

```markdown
### Fix Verification
1. Re-run the exact input that caused the incident
   - [ ] Output is now correct

2. Run the standard test suite
   - [ ] All tests pass

3. Run 5 similar inputs (edge cases, variations)
   - [ ] All produce correct output

4. Run a full regression (if P0 or P1)
   - [ ] No regressions detected

5. Monitor for 24 hours after deploying the fix
   - [ ] No recurrence observed
```

---

## Phase 5: Prevent Recurrence

**Goal**: Ensure this specific failure and this class of failure cannot happen again.

### 5.1 Update Guardrails

If the incident revealed a missing guardrail, add it:

```markdown
### New Guardrail
- Trigger: [what situation triggers this guardrail]
- Rule: [what the agent must or must not do]
- Enforcement: [how the harness enforces this]
- Added because: [link to incident]
```

### 5.2 Add Automated Detection

Add a check that would have caught this incident earlier:

```markdown
### New Automated Check
- Name: [descriptive name]
- Type: [pre-execution / post-execution / monitoring]
- Implementation: [how it works]
- Alert: [who gets notified and how]
- Added because: [link to incident]
```

### 5.3 Update Evaluation Criteria

If the incident revealed a gap in evaluation:

```markdown
### Updated Evaluation
- New criterion: [description]
- Measurement: [how to measure]
- Target: [acceptable threshold]
- Added because: [link to incident]
```

### 5.4 Write a Lesson Learned

Add to `tasks/lessons.md` (or equivalent):

```markdown
### Lesson: [Title]
**Date**: [date]
**Incident**: [brief description]
**Root Cause**: [root cause in one sentence]
**Lesson**: [What we learned, written as a rule for future reference]
**Action Taken**: [What we changed to prevent recurrence]
```

### 5.5 Conduct a Retrospective (for P0 and P1)

For critical incidents, hold a structured retrospective:

```markdown
## Incident Retrospective: [Title]

### Timeline
[Chronological account of what happened]

### What Went Well
- [Detection was fast because...]
- [Containment was effective because...]

### What Went Wrong
- [Root cause was X]
- [Detection was slow because...]
- [Containment was incomplete because...]

### Action Items
- [ ] [Specific, assignable, time-bound action]
- [ ] [Specific, assignable, time-bound action]

### Systemic Issues
[Are there broader patterns this incident reveals?]
```

### Prevention Checklist

- [ ] Guardrails updated (if applicable)
- [ ] Automated detection added for this failure mode
- [ ] Evaluation criteria updated (if applicable)
- [ ] Lesson learned documented
- [ ] Retrospective completed (if P0 or P1)
- [ ] Similar code paths reviewed for the same class of issue
- [ ] Team notified of the new guardrail/check

---

## Quick Reference: Incident Response Flow

```
DETECT
  |
  +--> Classify severity (P0-P3)
  |
  v
CONTAIN (act first, investigate later)
  |
  +--> Stop agent if running
  +--> Revert external changes (P0/P1)
  +--> Preserve logs
  |
  v
DIAGNOSE (understand before fixing)
  |
  +--> Gather evidence
  +--> Walk through execution trace
  +--> Identify root cause
  +--> Reproduce
  |
  v
FIX (root cause, not symptom)
  |
  +--> Implement minimal fix
  +--> Test against failing input
  +--> Regression test
  |
  v
PREVENT RECURRENCE
  |
  +--> Update guardrails
  +--> Add automated detection
  +--> Document lesson learned
  +--> Retrospective (P0/P1)
```

---

## Appendix: Common Incidents and Quick Fixes

| Incident | Quick Diagnosis | Quick Fix |
|----------|----------------|-----------|
| Agent loops infinitely | Missing or too-high iteration limit | Add/lower max iterations in harness |
| Agent calls wrong tool | Ambiguous tool descriptions | Clarify tool descriptions; restrict available tools per step |
| Output contains hallucinated data | No grounding in provided context | Add "only use provided data" constraint; add fact-check step |
| Agent ignores instructions | Instructions buried in long context | Move critical instructions to start/end of prompt; reduce context |
| Agent modifies wrong files | No scope restriction | Add file whitelist to task definition |
| Excessive token usage | Large context, verbose output | Summarize context; add output length constraints |
| Tool authentication failure | Token expired or misconfigured | Refresh token; check credential storage |
| Agent produces empty output | Prompt too constraining or context too large | Simplify constraints; reduce context; check for token limit overflow |
