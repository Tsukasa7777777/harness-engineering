# Harness Specification

**Version**: 1.0.0  
**Status**: Draft  
**Last Updated**: 2026-03-31

## 1. Overview

A harness is the complete orchestration layer around a foundation model that transforms it from a raw capability into a production system. This specification defines the components, interfaces, and behaviors of a well-engineered harness.

## 2. Architecture

```
+---------------------------------------------+
|                  Harness                     |
|                                             |
|  +---------+  +----------+  +-----------+   |
|  | Context |  |Constraints|  |   Tools   |  |
|  | Manager |  |  Engine   |  |  Router   |  |
|  +----+----+  +-----+----+  +-----+-----+  |
|       |              |              |        |
|  +----v--------------v--------------v----+   |
|  |          Prompt Assembler              |  |
|  +-----------------+---------------------+   |
|                    |                         |
|  +-----------------v---------------------+   |
|  |         Foundation Model               |  |
|  +-----------------+---------------------+   |
|                    |                         |
|  +-----------------v---------------------+   |
|  |         Output Processor               |  |
|  |  (validation, formatting, routing)     |  |
|  +---------------------------------------+   |
|                                             |
|  +---------------------------------------+   |
|  |         Evaluation Engine              |  |
|  |  (quality metrics, regression tests)  |   |
|  +---------------------------------------+   |
+---------------------------------------------+
```

## 3. Components

### 3.1 Context Manager

**Responsibility**: Assembles the context window from multiple sources with priority ordering and token budgeting.

**Inputs**:
- Agent identity (from `01_context/`)
- Conversation history
- Tool results
- Dynamic context (retrieved documents, database records)

**Behavior**:
- Allocates token budgets per section
- Truncates lower-priority content when approaching limits
- Caches frequently used context fragments
- Tracks context utilization metrics

### 3.2 Constraints Engine

**Responsibility**: Enforces guardrails, tool policies, and escalation rules before and after model inference.

**Pre-execution checks**:
- Input validation against blocklists
- Tool authorization verification
- Rate limit enforcement

**Post-execution checks**:
- Output content validation
- Format compliance
- Escalation trigger evaluation

### 3.3 Tools Router

**Responsibility**: Maps agent tool requests to MCP servers and manages tool execution lifecycle.

**Behavior**:
- Resolves tool names to MCP server endpoints
- Applies tool-specific policies (rate limits, approval gates)
- Handles tool errors with retries and fallbacks
- Logs tool usage for audit and evaluation

### 3.4 Prompt Assembler

**Responsibility**: Constructs the final prompt from context, constraints, and current task.

**Assembly order** (highest to lowest priority):
1. System identity and role
2. Hard constraints and guardrails
3. Task-specific instructions
4. Relevant context and examples
5. Conversation history (truncated as needed)
6. Current user input

### 3.5 Output Processor

**Responsibility**: Validates, formats, and routes model output.

**Processing pipeline**:
1. Parse structured output (JSON, YAML, code blocks)
2. Validate against schema (if defined)
3. Apply formatting rules
4. Route to destination (user, tool, sub-agent)

### 3.6 Evaluation Engine

**Responsibility**: Continuously measures harness quality against defined benchmarks.

**Capabilities**:
- Automated eval execution on every configuration change
- Regression detection across versions
- Quality metric tracking over time
- A/B comparison between harness configurations

## 4. Lifecycle

### 4.1 Request Lifecycle

```
User Input
    |
    v
Pre-execution Constraints Check
    |
    v
Context Assembly (token budgeting)
    |
    v
Prompt Assembly
    |
    v
Model Inference
    |
    v
Output Processing
    |
    v
Post-execution Constraints Check
    |
    v
Response Delivery (or Escalation)
```

### 4.2 Configuration Lifecycle

```
Edit Configuration
    |
    v
Run Eval Suite
    |
    v
Compare with Baseline
    |
    v
Review Regressions (if any)
    |
    v
Approve and Deploy
    |
    v
Monitor Production Metrics
```

## 5. Quality Attributes

| Attribute | Target | Measurement |
|-----------|--------|-------------|
| **Latency** | < 2s for context assembly | P95 timing metric |
| **Accuracy** | > 90% on domain-specific evals | Eval pass rate |
| **Safety** | 0 guardrail violations in production | Violation count |
| **Reliability** | > 99.5% successful completions | Error rate |
| **Observability** | 100% of tool calls logged | Audit coverage |

## 6. Acceptance Criteria

A harness implementation satisfies this spec when:

- [ ] All six components are implemented and operational
- [ ] Context window usage is tracked and budgeted
- [ ] Guardrails prevent all defined constraint violations
- [ ] Tool calls are logged with request/response pairs
- [ ] Eval suite passes with > 90% accuracy on core benchmarks
- [ ] Configuration changes trigger automated eval runs
