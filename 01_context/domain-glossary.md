# Domain Glossary

## Overview

A shared vocabulary ensures the agent and users communicate precisely. This glossary defines domain-specific terms used throughout the harness engineering framework.

## Core Concepts

### Agent Harness
The orchestration layer that wraps a foundation model with context, constraints, tools, evaluations, and workflows. The harness is everything *around* the model that makes it useful in production.

### Context Window
The total input provided to the model in a single inference call, including system prompts, conversation history, tool results, and user input. Managing context window usage is a critical engineering challenge.

### Foundation Model
The underlying large language model (e.g., Claude, GPT, Gemini) that the harness orchestrates. The harness treats the model as a capability to be managed, not a product to be shipped directly.

### Guardrail
A constraint applied to agent behavior that prevents harmful, incorrect, or unauthorized actions. Guardrails can be pre-execution (input filtering), post-execution (output validation), or runtime (tool-use policies).

### MCP (Model Context Protocol)
An open protocol for connecting AI models to external tools and data sources. MCP servers expose capabilities that agents can invoke through a standardized interface.

### Prompt Architecture
The structured design of system prompts, including section ordering, formatting conventions, and information hierarchy. Good prompt architecture maximizes the model's ability to follow instructions.

### Tool Policy
A set of rules governing how and when an agent can use a specific tool, including rate limits, approval requirements, and scope restrictions.

## Workflow Terminology

### Delegation Pattern
A workflow where a primary agent assigns subtasks to specialized sub-agents, each optimized for a specific capability.

### Escalation
The process of transferring a task from an automated agent to a human operator when the task exceeds the agent's confidence or authority thresholds.

### Iterative Refinement
A workflow pattern where the agent produces an initial output, evaluates it against criteria, and improves it through multiple passes.

### Runbook
A step-by-step procedure for handling a specific scenario, designed to be followed by an agent or human operator with minimal judgment required.

### Sequential Workflow
A workflow where steps execute in a fixed order, with each step's output feeding into the next step's input.

## Evaluation Terminology

### Benchmark
A standardized test suite that measures agent performance on a specific capability, enabling comparison across versions and configurations.

### Eval (Evaluation)
A test that measures whether an agent's output meets defined quality criteria. Evals can be automated (deterministic checks) or human-judged (qualitative assessment).

### Regression Test
An eval that verifies previously working behavior has not degraded after a change to the harness configuration.

### Ground Truth
The known-correct answer or behavior for an eval case, used as the reference for measuring agent accuracy.

## Infrastructure Terminology

### Sandbox
An isolated execution environment where agent-invoked tools operate with restricted permissions, preventing unintended side effects on production systems.

### Context Injection
The process of dynamically adding relevant information to the context window based on the current task, such as retrieving documentation or database records.

### Token Budget
The allocated portion of the context window for a specific purpose (e.g., system prompt, conversation history, tool results). Managing token budgets prevents context overflow.

---

*Add new terms as they emerge. Keep definitions concise and example-oriented.*
