# Failure Modes in Agent Systems

**Last Updated**: 2026-03-31

## Overview

Understanding common failure modes is essential for building resilient agent systems. This document catalogs known failure patterns, their symptoms, root causes, and mitigations.

## Failure Catalog

### F1: Context Window Overflow

**Symptom**: Agent starts ignoring instructions, producing incoherent output, or "forgetting" earlier conversation.

**Root Cause**: Context window is full, causing critical information (system prompt, guardrails) to be truncated.

**Impact**: High — can cause guardrail violations, task failure, or unpredictable behavior.

**Mitigation**:
- Implement token budgeting with priority-based truncation
- Monitor context utilization and alert at 85%+
- Never truncate system identity or guardrail sections
- Use conversation summarization for long sessions

---

### F2: Tool Misselection

**Symptom**: Agent calls the wrong tool, or calls a tool when no tool is needed.

**Root Cause**: Tool descriptions are ambiguous, overlapping, or the agent misunderstands the user's intent.

**Impact**: Medium — can cause incorrect actions, wasted API calls, or user confusion.

**Mitigation**:
- Write clear, non-overlapping tool descriptions
- Include negative examples ("Do NOT use this tool for X")
- Test tool selection with the B2 benchmark suite
- Add a confirmation step for high-consequence tools

---

### F3: Infinite Loop

**Symptom**: Agent repeatedly calls the same tool or asks the same clarifying question without making progress.

**Root Cause**: No termination condition, or the agent cannot recognize that it is stuck.

**Impact**: High — wastes resources, blocks the user, can exhaust rate limits.

**Mitigation**:
- Set maximum iteration counts for all loops
- Implement "stuck detection" (same tool call with same params N times)
- Add escalation after N failed attempts
- Log loop count as a monitoring metric

---

### F4: Hallucinated Tool Calls

**Symptom**: Agent attempts to call tools that do not exist, or passes parameters with fabricated values.

**Root Cause**: Model generates plausible-sounding but non-existent tool names, or fills in required parameters with guesses instead of asking the user.

**Impact**: Medium — causes errors, potentially dangerous if the fabricated call resembles a real one.

**Mitigation**:
- Validate tool names against the registered tool list before execution
- Validate parameters against tool schemas
- Train the agent to say "I do not have a tool for that" when appropriate
- Log and review hallucinated tool calls

---

### F5: Guardrail Bypass

**Symptom**: Agent performs an action that should have been blocked by constraints.

**Root Cause**: Prompt injection, context window overflow pushing guardrails out, or insufficient constraint specification.

**Impact**: Critical — can cause data loss, security breaches, or compliance violations.

**Mitigation**:
- Place guardrails at the highest priority in context window
- Implement both prompt-level AND code-level guardrails
- Run adversarial evals (B3 benchmark) regularly
- Monitor for constraint violations in production

---

### F6: Cascading Failure

**Symptom**: One failing component causes a chain of failures across the system.

**Root Cause**: Tight coupling between components, no error isolation, missing fallback strategies.

**Impact**: High — can take down the entire agent system.

**Mitigation**:
- Implement circuit breakers for tool calls
- Use graceful degradation (continue without failed tool)
- Isolate sub-agent failures from the orchestrator
- Have fallback responses for common failure scenarios

---

### F7: Drift

**Symptom**: Agent behavior gradually changes over time without any harness configuration changes.

**Root Cause**: Model updates by the provider, accumulated conversation context, or changing user patterns.

**Impact**: Medium — subtle quality degradation that is hard to detect.

**Mitigation**:
- Run regression tests on a schedule (weekly minimum)
- Pin model versions where possible
- Monitor eval scores as a time series
- Compare production behavior against baseline regularly

---

### F8: Confidential Data Leakage

**Symptom**: Agent includes sensitive information (API keys, customer data, internal details) in responses.

**Root Cause**: Sensitive data present in context window, no output filtering, or model trained on sensitive data.

**Impact**: Critical — security breach, compliance violation, reputational damage.

**Mitigation**:
- Minimize sensitive data in context (retrieve only what is needed)
- Implement output scanning for known sensitive patterns
- Mark sensitive context sections explicitly
- Audit outputs regularly for data leakage

---

### F9: Inconsistent Persona

**Symptom**: Agent switches between different communication styles, expertise levels, or personalities.

**Root Cause**: Weak persona definition, conflicting instructions, or context window pressure causing identity truncation.

**Impact**: Low-Medium — erodes user trust and confuses expectations.

**Mitigation**:
- Place identity definition at highest priority in context
- Include concrete examples of desired communication style
- Test persona consistency with paired eval cases
- Monitor user feedback for consistency complaints

---

### F10: Over-Escalation

**Symptom**: Agent escalates too many requests to humans, defeating the purpose of automation.

**Root Cause**: Escalation thresholds too conservative, agent uncertainty too high, or insufficient capability for the task.

**Impact**: Medium — reduces automation value, creates bottleneck at human operators.

**Mitigation**:
- Tune escalation thresholds based on production data
- Track escalation rate as a key metric
- Analyze escalated requests to identify capability gaps
- Provide the agent with more context to reduce uncertainty

## Failure Response Matrix

| Failure Mode | Detection | Automated Response | Human Response |
|-------------|-----------|-------------------|----------------|
| F1: Context Overflow | Token count monitoring | Truncate low-priority sections | Review budget allocation |
| F2: Tool Misselection | Tool call validation | Retry with clarification | Update tool descriptions |
| F3: Infinite Loop | Iteration counter | Break and escalate | Debug root cause |
| F4: Hallucinated Tools | Schema validation | Reject and retry | Review tool documentation |
| F5: Guardrail Bypass | Output scanning | Block response, alert | Investigate and patch |
| F6: Cascading Failure | Health checks | Circuit breaker | Root cause analysis |
| F7: Drift | Regression tests | Alert on score drop | Re-evaluate configuration |
| F8: Data Leakage | Output scanning | Redact and alert | Security review |
| F9: Persona Drift | Style analysis | Reset context | Strengthen identity prompt |
| F10: Over-Escalation | Escalation rate | Adjust thresholds | Capability assessment |
