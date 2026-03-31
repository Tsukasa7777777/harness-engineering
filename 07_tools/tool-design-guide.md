# Tool Design Guide

This guide defines how to design tools that AI agents can invoke reliably, safely, and efficiently. These guidelines apply whether you are wrapping an existing API as an MCP tool, building a custom tool, or evaluating a third-party MCP server.

The core insight: **an agent reads the tool's name, description, and parameter schema to decide when and how to use it**. If any of these are ambiguous, the agent will misuse the tool. Design for the agent as your primary user.

---

## Table of Contents

1. [Naming Conventions](#1-naming-conventions)
2. [Parameter Design](#2-parameter-design)
3. [Error Messages](#3-error-messages)
4. [Idempotency](#4-idempotency)
5. [Output Format](#5-output-format)
6. [Documentation Requirements](#6-documentation-requirements)
7. [Tool Granularity](#7-tool-granularity)
8. [Anti-Patterns](#8-anti-patterns)

---

## 1. Naming Conventions

Tool names are the first thing an agent reads when deciding which tool to use. Names must be unambiguous and self-explanatory.

### Rules

- Use **verb-noun** format: the verb describes the action, the noun describes the target.
- Use **snake_case** for multi-word names.
- Be specific enough to distinguish from similar tools.
- Avoid generic verbs like `do`, `run`, `process`, `handle` unless the tool is genuinely generic.
- Prefix with a namespace if multiple tools share a domain (e.g., `github_create_issue`, `github_list_repos`).

### Examples

| Bad | Good | Why |
|---|---|---|
| `get_data` | `fetch_user_profile` | "data" is meaningless; specify what data |
| `do_thing` | `send_email` | Be explicit about the action |
| `update` | `update_spreadsheet_cell` | Specify the target |
| `github` | `github_create_issue` | Include the action, not just the domain |
| `emailTool` | `search_emails` | Use verb-noun, not domain+suffix |
| `handleCalendar` | `list_calendar_events` | "handle" is vague; specify the operation |
| `create_or_update_record` | `create_record` / `update_record` | Split into separate tools (see Granularity) |

---

## 2. Parameter Design

Parameters are the agent's interface for passing data to the tool. Poor parameter design is the number one cause of tool misuse.

### Rules

- **Explicit types**: Always declare the type (`string`, `number`, `boolean`, `array`, `object`). Never accept `any`.
- **Required vs optional**: Mark parameters as required only if the tool cannot function without them. Provide sensible defaults for optional parameters.
- **Good defaults**: If 90% of calls use the same value, make it the default.
- **Constrained values**: Use enums for parameters with a fixed set of valid values.
- **Descriptive names**: Parameter names should describe the content, not the format.
- **No boolean traps**: Avoid boolean parameters whose meaning is unclear at the call site. Use enums or separate tools instead.
- **Flat over nested**: Prefer flat parameter lists over deeply nested objects. Agents handle flat schemas more reliably.

### Examples

**Bad: Ambiguous types and names**
```json
{
  "name": "query",
  "data": "something",
  "flag": true
}
```

**Good: Explicit types and descriptive names**
```json
{
  "search_query": {
    "type": "string",
    "description": "Text to search for in email subjects and bodies",
    "required": true
  },
  "max_results": {
    "type": "number",
    "description": "Maximum number of emails to return (1-100)",
    "default": 10,
    "required": false
  },
  "include_spam": {
    "type": "boolean",
    "description": "Whether to include emails from the spam folder",
    "default": false,
    "required": false
  }
}
```

**Bad: Boolean trap**
```json
{
  "create_issue": {
    "params": {
      "private": true
    }
  }
}
```
Does `private: true` mean the issue is private, or that it uses a private repository? Ambiguous.

**Good: Enum instead of boolean**
```json
{
  "create_issue": {
    "params": {
      "visibility": {
        "type": "string",
        "enum": ["public", "private", "internal"],
        "default": "public",
        "description": "Who can see this issue"
      }
    }
  }
}
```

**Bad: Deeply nested parameters**
```json
{
  "config": {
    "output": {
      "format": {
        "type": "json",
        "options": {
          "pretty": true
        }
      }
    }
  }
}
```

**Good: Flat parameters**
```json
{
  "output_format": {
    "type": "string",
    "enum": ["json", "csv", "text"],
    "default": "json"
  },
  "pretty_print": {
    "type": "boolean",
    "default": true
  }
}
```

---

## 3. Error Messages

When a tool fails, the error message is the agent's only clue for what went wrong and what to do next. Errors must be **actionable**.

### Rules

- **State what happened**: Describe the failure, not just a status code.
- **State why it happened**: Include the root cause if known.
- **State how to fix it**: Provide concrete recovery steps.
- **Include context**: Echo back the relevant input that caused the error.
- **Use consistent structure**: All errors should follow the same format.
- **Never return empty errors**: An empty string or `null` error is worse than a wrong error.

### Error Format Template

```
Error: [WHAT happened]
Cause: [WHY it happened]
Input: [RELEVANT input that caused the error]
Recovery: [HOW to fix it]
```

### Examples

**Bad: Opaque error**
```
Error: 404
```

**Bad: Error without recovery**
```
Error: Authentication failed
```

**Good: Actionable error**
```
Error: Failed to create GitHub issue in repository "walkers-inc/frontend"
Cause: Authentication token has expired (HTTP 401)
Input: owner="walkers-inc", repo="frontend", title="Fix login bug"
Recovery: Refresh the GitHub Personal Access Token. Check that the token has "repo" scope. Token location: .env GITHUB_TOKEN
```

**Good: Input validation error**
```
Error: Invalid parameter "max_results"
Cause: Value 500 exceeds the maximum allowed value of 100
Input: max_results=500
Recovery: Set max_results to a value between 1 and 100. Use pagination (page_token) to retrieve more results.
```

**Good: Rate limit error**
```
Error: Rate limit exceeded for GitHub API
Cause: 5,000 requests per hour limit reached. Resets at 2025-01-15T14:30:00Z.
Input: github_list_issues (this was the 5,001st request)
Recovery: Wait until 2025-01-15T14:30:00Z (approximately 12 minutes) before retrying. Consider batching requests or caching results.
```

---

## 4. Idempotency

An idempotent tool produces the same result whether called once or multiple times with the same inputs. This is critical because agents may retry failed operations.

### Rules

- **Read operations**: Must always be idempotent. `list_events`, `get_user`, `search_emails` should never modify state.
- **Create operations**: Use a client-generated idempotency key (e.g., UUID) when the underlying API supports it. Document whether the tool is idempotent.
- **Update operations**: Prefer "set to value X" over "increment by Y". Absolute updates are idempotent; relative updates are not.
- **Delete operations**: Should succeed silently if the target is already deleted (not error on "not found").
- **Document it**: Every tool's description must state whether it is idempotent.

### Examples

**Non-idempotent (dangerous for retries)**
```
Tool: increment_counter
Input: { "counter_name": "page_views", "amount": 1 }
// Calling twice increments by 2 -- unintended!
```

**Idempotent (safe for retries)**
```
Tool: set_counter
Input: { "counter_name": "page_views", "value": 42 }
// Calling twice still results in value 42
```

**Non-idempotent create (dangerous)**
```
Tool: create_issue
Input: { "title": "Fix bug", "body": "..." }
// Calling twice creates two duplicate issues
```

**Idempotent create (safe)**
```
Tool: create_issue
Input: { "title": "Fix bug", "body": "...", "idempotency_key": "fix-bug-2025-01-15" }
// Calling twice creates only one issue
```

---

## 5. Output Format

The tool's output is what the agent reads to decide its next action. Outputs must be structured, predictable, and parseable.

### Rules

- **Structured data**: Return JSON objects, not prose. Agents parse structured data more reliably.
- **Consistent schema**: Every successful call returns the same top-level keys. Do not change the schema based on the data.
- **Include metadata**: Return pagination info, total counts, and timestamps where relevant.
- **Limit output size**: If a result could be huge (e.g., listing thousands of items), paginate by default. Do not return unbounded lists.
- **Separate success and error**: Use distinct structures for success and failure. Do not embed errors in the data payload.
- **Human-readable summaries**: Include a `summary` field for complex outputs so the agent can quickly understand the result without parsing every field.

### Examples

**Bad: Unstructured text output**
```
Found 3 issues. The first one is about login bugs, created yesterday by Alice.
The second one is about performance, created last week...
```

**Good: Structured output**
```json
{
  "total_count": 3,
  "page": 1,
  "page_size": 10,
  "has_more": false,
  "items": [
    {
      "id": 1234,
      "title": "Login button unresponsive on mobile",
      "state": "open",
      "created_at": "2025-01-14T09:00:00Z",
      "author": "alice"
    },
    {
      "id": 1235,
      "title": "Page load time exceeds 3s on /dashboard",
      "state": "open",
      "created_at": "2025-01-08T15:30:00Z",
      "author": "bob"
    }
  ],
  "summary": "3 open issues found. Most recent: 'Login button unresponsive on mobile' (2025-01-14)."
}
```

**Bad: Inconsistent schema**
```json
// Sometimes returns:
{ "result": "success", "issue_id": 1234 }

// Other times returns:
{ "ok": true, "data": { "id": 1234 } }
```

**Good: Consistent schema**
```json
// Always returns:
{
  "success": true,
  "data": { "issue_id": 1234, "url": "https://github.com/..." },
  "summary": "Issue #1234 created successfully."
}
```

---

## 6. Documentation Requirements

Every tool must include documentation that allows an agent to use it correctly without prior experience. The documentation lives in the tool's schema, not in external files.

### Required Fields

| Field | Purpose | Example |
|---|---|---|
| `name` | Tool identifier (verb_noun format) | `search_emails` |
| `description` | What the tool does, when to use it, when NOT to use it | "Search Gmail for emails matching a query. Use this for finding specific emails. Do NOT use this for reading a known email by ID -- use get_email instead." |
| `parameters` | JSON schema with type, description, default, enum for each param | See Parameter Design section |
| `examples` | At least one example input/output pair | `{ "input": {...}, "output": {...} }` |
| `idempotency` | Whether the tool is safe to retry | "Idempotent: yes (read-only)" |
| `rate_limits` | Any rate limiting the agent should be aware of | "Max 3 requests/second" |
| `side_effects` | What state the tool modifies | "Creates a new issue in the specified repository" |

### Description Best Practices

The description is the most important field. It determines when the agent decides to use the tool.

**Bad description:**
```
"Manages emails"
```

**Good description:**
```
"Search for emails in Gmail matching a query string. Supports Gmail search syntax (e.g., 'from:alice@example.com is:unread after:2025/01/01'). Returns up to max_results emails sorted by date, newest first. Use this tool when you need to find emails by content, sender, date, or labels. Do NOT use this for reading a specific email when you already have its message ID -- use gmail_get_message instead. Do NOT use this for sending emails -- use gmail_send instead."
```

Key elements of a good description:
1. What it does (search emails)
2. How to use the input (Gmail search syntax)
3. What it returns (emails sorted by date)
4. When to use it (finding emails by criteria)
5. When NOT to use it (redirects to other tools)

---

## 7. Tool Granularity

How broadly or narrowly should you define a tool? This is the most consequential design decision.

### Rules

- **One tool, one action**: A tool should do exactly one thing. If you find yourself writing "or" in the description, split it into two tools.
- **Prefer many simple tools over few complex tools**: Agents are better at choosing from a list of specific tools than at configuring a complex multi-purpose tool.
- **Group by resource, not by UI**: `create_issue`, `update_issue`, `list_issues` -- not `manage_issues`.
- **Combine only when atomic**: If two actions must always happen together (e.g., upload file then get URL), combine them into one tool.

### Examples

**Bad: One tool does too much**
```
Tool: manage_project
Params: action ("create" | "delete" | "update" | "list" | "archive")
// Agent must figure out which action to use AND how the params change per action
```

**Good: Separate tools per action**
```
Tool: create_project    -- params: name, description
Tool: delete_project    -- params: project_id
Tool: update_project    -- params: project_id, name?, description?
Tool: list_projects     -- params: status?, page_size?
Tool: archive_project   -- params: project_id
```

**Acceptable combination: Atomic operation**
```
Tool: upload_and_get_url
// Upload a file and return its public URL. These two steps are always needed together.
```

---

## 8. Anti-Patterns

### Anti-Pattern 1: The Kitchen Sink Tool

A single tool that accepts dozens of parameters and performs different operations based on flags.

**Problem**: The agent cannot reliably determine which parameters to set for which operation. Leads to hallucinated parameter combinations.

**Fix**: Split into focused tools.

### Anti-Pattern 2: Silent Failure

A tool returns a success status but silently drops data or skips operations.

**Problem**: The agent believes the operation succeeded and moves on, but the actual state is wrong.

**Fix**: Return explicit errors for any partial failure. Use the error format template.

### Anti-Pattern 3: Ambiguous Parameter Names

Parameters named `data`, `input`, `value`, `config`, `options`, `args`, `params`.

**Problem**: The agent does not know what to pass because the name gives no semantic information.

**Fix**: Name parameters after their content: `email_address`, `search_query`, `repository_name`.

### Anti-Pattern 4: Unstructured Output

Returning a prose paragraph instead of structured data.

**Problem**: The agent must parse natural language, which is unreliable and wastes tokens.

**Fix**: Return JSON with consistent top-level keys.

### Anti-Pattern 5: Missing When-Not-To-Use Guidance

A description that only says what the tool does, but not when to choose a different tool.

**Problem**: The agent uses this tool in situations where another tool would be better, because it cannot distinguish the two.

**Fix**: Include "Do NOT use this when..." in the description, with a pointer to the correct tool.

### Anti-Pattern 6: God Tool with Subcommands

A tool named `execute` or `run` that takes a `command` parameter to specify what to do.

**Problem**: This is essentially a custom DSL. The agent must know the command vocabulary, which it can only learn from examples.

**Fix**: Expose each command as a separate tool with its own schema.

### Anti-Pattern 7: Non-Deterministic Defaults

Default values that depend on time, random state, or external conditions without documenting this.

**Problem**: The agent cannot predict what will happen when it omits a parameter.

**Fix**: Make defaults deterministic. If a default depends on context (e.g., "current date"), document it explicitly.

---

## Checklist: Before Shipping a New Tool

- [ ] Name follows verb_noun convention
- [ ] Description explains what, when, and when-not-to-use
- [ ] All parameters have explicit types and descriptions
- [ ] Required vs optional is correctly marked
- [ ] Sensible defaults for optional parameters
- [ ] At least one input/output example in documentation
- [ ] Error messages include cause and recovery steps
- [ ] Idempotency behavior is documented
- [ ] Output is structured JSON with consistent schema
- [ ] Output is bounded (pagination for lists)
- [ ] Side effects are documented
- [ ] Integration test exists in `integration-tests.md`
- [ ] Registered in `mcp-registry.md`
