# Prompt Architecture

**Version**: 1.0.0  
**Status**: Draft  
**Last Updated**: 2026-03-31

## Overview

Prompt architecture defines the structure, layering, and formatting standards for system prompts. A well-architected prompt is the difference between an agent that follows instructions reliably and one that drifts off-task.

## Layered Prompt Structure

### Layer Model

```
+------------------------------------+
|  Layer 1: Identity & Role          |  <- Who am I?
+------------------------------------+
|  Layer 2: Constraints & Rules      |  <- What must I never/always do?
+------------------------------------+
|  Layer 3: Capabilities & Tools     |  <- What can I do?
+------------------------------------+
|  Layer 4: Domain Knowledge         |  <- What do I know?
+------------------------------------+
|  Layer 5: Task Instructions        |  <- What am I doing right now?
+------------------------------------+
|  Layer 6: Examples & Few-Shot      |  <- How should I do it?
+------------------------------------+
|  Layer 7: Context & History        |  <- What happened before?
+------------------------------------+
```

### Layer Descriptions

| Layer | Content | Mutability | Priority |
|-------|---------|-----------|----------|
| **1. Identity** | Agent name, role, personality | Static | Highest |
| **2. Constraints** | Guardrails, policies, boundaries | Static | Highest |
| **3. Capabilities** | Available tools, their descriptions | Semi-static | High |
| **4. Domain Knowledge** | Glossary, reference data, FAQs | Semi-static | Medium |
| **5. Task Instructions** | Current task description, requirements | Dynamic | High |
| **6. Examples** | Few-shot examples, output templates | Semi-static | Medium |
| **7. Context** | Conversation history, tool results | Dynamic | Lower |

## Formatting Standards

### Section Headers

Use clear, hierarchical headers to help the model parse prompt structure:

```markdown
# Role
You are a senior software engineer...

# Rules
## Hard Rules (Never Violate)
- Never execute destructive git commands...

## Soft Rules (Prefer Unless Instructed Otherwise)
- Prefer concise responses...

# Available Tools
## GitHub
- Create issues, pull requests...

# Current Task
...
```

### Emphasis Patterns

| Pattern | Usage | Example |
|---------|-------|---------|
| `**bold**` | Key terms and concepts | **Never** commit secrets |
| `UPPERCASE` | Critical warnings | IMPORTANT: This will delete data |
| `> blockquote` | Rules and constraints | > Always ask before deploying |
| backtick code | Technical terms, commands | Use `git rebase` instead of merge |

### Lists vs. Prose

- **Use lists** for rules, constraints, and capabilities (easier for models to parse)
- **Use prose** for role descriptions, context, and nuanced instructions
- **Use tables** for structured data with multiple attributes

## Anti-Patterns

### 1. Wall of Text
**Problem**: Long, unstructured paragraphs that bury important instructions.  
**Fix**: Break into sections with clear headers and use lists for discrete items.

### 2. Contradictory Instructions
**Problem**: Different sections give conflicting guidance.  
**Fix**: Establish a priority hierarchy (Layer 2 constraints override Layer 5 task instructions).

### 3. Implicit Expectations
**Problem**: Assuming the model will infer unstated requirements.  
**Fix**: Be explicit. If you want JSON output, say "Respond in JSON format."

### 4. Over-Prompting
**Problem**: Including so many instructions that the model cannot prioritize.  
**Fix**: Focus on the 20% of instructions that drive 80% of quality. Move edge cases to separate documents.

### 5. Under-Specifying Output Format
**Problem**: Getting inconsistent response formats across invocations.  
**Fix**: Provide a concrete output template or schema.

## Template

```markdown
# Identity
You are [role]. You help [audience] with [primary tasks].

# Rules
## Critical (Never Violate)
- [Rule 1]
- [Rule 2]

## Default Behaviors (Override with User Instructions)
- [Preference 1]
- [Preference 2]

# Tools
You have access to the following tools:
- **[Tool Name]**: [Brief description of when and how to use it]

# Output Format
Respond using this structure:
[Template or schema]

# Current Task
[Dynamic task context inserted here]
```

## Validation Checklist

Before deploying a prompt:

- [ ] Identity is specific and actionable (not vague)
- [ ] Constraints are organized by severity (critical vs. preferred)
- [ ] All available tools are listed with usage guidance
- [ ] Output format is explicitly specified
- [ ] No contradictions between layers
- [ ] Total token count is within budget
- [ ] Tested against eval suite with passing scores
