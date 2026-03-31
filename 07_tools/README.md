# 07 Tools: Tool Integration & MCP Configuration

## Purpose

This section defines how AI agents discover, invoke, and manage external tools. In harness engineering, tools are the bridge between an agent's reasoning and the real world -- they turn plans into actions. A well-designed tool layer means agents can operate autonomously, handle errors gracefully, and scale to new capabilities without rewriting core logic.

## What is MCP?

The **Model Context Protocol (MCP)** is an open standard for connecting AI agents to external data sources and tools. It provides:

- **Standardized interface**: A uniform way for agents to discover and invoke tools regardless of the underlying service (GitHub, databases, file systems, APIs).
- **Server/client architecture**: MCP servers expose tools; the agent (client) discovers and calls them via a well-defined protocol.
- **Dynamic tool discovery**: Agents can enumerate available tools, read their schemas, and decide at runtime which tools to use.
- **Transport flexibility**: Supports stdio, SSE (Server-Sent Events), and HTTP transports.

Think of MCP as "USB for AI tools" -- plug in a new server and the agent immediately knows what it can do.

## Why Tools Matter in Harness Engineering

Without tools, an agent is limited to generating text. With tools, an agent can:

| Capability | Example |
|---|---|
| Read external data | Query a database, fetch a web page, read a file |
| Write to external systems | Create a GitHub issue, send an email, update a spreadsheet |
| Execute code | Run tests, deploy services, analyze data |
| Interact with users | Schedule meetings, post messages, manage tasks |
| Observe the world | Monitor errors, track metrics, search the web |

The quality of your tool layer directly determines the ceiling of your agent's capabilities. Poor tool design leads to confused agents, wasted tokens, and failed tasks.

## Files in This Section

| File | Purpose |
|---|---|
| [`mcp-registry.md`](./mcp-registry.md) | Registry of all MCP servers -- what's connected, how it's authenticated, and current status |
| [`tool-design-guide.md`](./tool-design-guide.md) | Guidelines for designing tools that agents can invoke reliably and safely |
| [`integration-tests.md`](./integration-tests.md) | Test definitions to verify that each tool/MCP connection works correctly |

## How to Use This Section

### When adding a new tool

1. **Design the tool** using [`tool-design-guide.md`](./tool-design-guide.md) -- follow naming, parameter, and error conventions.
2. **Register it** in [`mcp-registry.md`](./mcp-registry.md) -- record the package, auth method, rate limits, and status.
3. **Write integration tests** in [`integration-tests.md`](./integration-tests.md) -- define at least one smoke test per tool.
4. **Run the tests** to verify the connection works end-to-end.

### When debugging a tool failure

1. Check [`mcp-registry.md`](./mcp-registry.md) -- is the tool still connected? Has auth expired?
2. Check [`integration-tests.md`](./integration-tests.md) -- run the relevant test to isolate the failure.
3. Check the tool's error output against the patterns in [`tool-design-guide.md`](./tool-design-guide.md).

### When reviewing agent behavior

If an agent is misusing a tool or failing to use one, consult [`tool-design-guide.md`](./tool-design-guide.md) to evaluate whether the tool's interface is the root cause. Common issues:

- Ambiguous parameter names cause the agent to pass wrong values.
- Missing error messages cause the agent to retry indefinitely.
- Overly broad tools cause the agent to choose the wrong one.

## Key Principles

1. **Tools should be atomic**: One tool does one thing well. Compose complex operations from simple tools.
2. **Tools should be self-describing**: The name, description, and parameter schema should be sufficient for an agent to use the tool correctly without external documentation.
3. **Tools should fail loudly**: Return clear, actionable error messages. Never silently succeed with partial results.
4. **Tools should be idempotent where possible**: Calling the same tool with the same inputs twice should not cause unintended side effects.
5. **Tools should be testable**: Every tool needs at least one integration test that can be run to verify connectivity.

## Further Reading

- Anthropic: "Building effective agents" -- tool use patterns
- Anthropic: "Writing effective tools for agents" -- design guidelines
- Model Context Protocol specification: https://modelcontextprotocol.io
- MCP server registry: https://github.com/modelcontextprotocol/servers
