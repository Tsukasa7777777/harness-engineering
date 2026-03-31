# 12 Factor Agents Checklist

A compliance checklist based on the 12 Factor Agents principles. Use this when designing, reviewing, or auditing an agent system. Each factor includes a description, rationale, compliance check, and implementation notes.

---

## Factor 1: Natural Language as the Universal Interface

### Description
Natural language is the protocol between humans and agents, and between agents and other agents. Inputs, outputs, instructions, and inter-agent communication all flow through natural language.

### Why It Matters
Natural language is the only interface that does not require the human (or calling agent) to learn a new API. It makes agents composable: any agent can call any other agent by describing what it needs in plain text. It also makes the system inspectable -- you can read the communication log and understand what happened.

### Compliance Check
- [ ] Agent accepts natural language task descriptions as input
- [ ] Agent produces natural language summaries alongside structured output
- [ ] Inter-agent communication uses natural language (not proprietary message formats)
- [ ] Prompts are readable by a non-technical stakeholder
- [ ] Error messages are human-readable, not raw stack traces

### Implementation Notes
- Structured output (JSON, tool calls) is a subset of natural language, not a replacement. Use structured output for machine-consumed data; use natural language for everything else.
- When agents communicate, include both a structured payload and a natural language explanation. The explanation aids debugging.

---

## Factor 2: Put Your Agent in a Harness

### Description
The agent does not run autonomously. It runs inside a harness -- your application code that manages the loop, controls tool execution, handles errors, and decides when to stop.

### Why It Matters
Without a harness, the LLM controls everything: what tools to call, when to stop, how to handle errors. This is fragile and unobservable. The harness gives you deterministic control over the non-deterministic LLM.

### Compliance Check
- [ ] A harness (your code) wraps every LLM call
- [ ] The harness decides when to loop, branch, or terminate
- [ ] Tool execution happens in the harness, not inside the LLM
- [ ] The harness enforces timeouts and iteration limits
- [ ] The harness logs every step for observability

### Implementation Notes
- The simplest harness is a `while` loop that calls the LLM, checks the response, executes any tool calls, and decides whether to continue.
- Frameworks like LangChain or CrewAI provide harness primitives. Use them, but understand what they abstract.

---

## Factor 3: Own Your Prompts

### Description
Prompts are first-class code artifacts. They live in your repository, are version-controlled, reviewed, and tested like any other code.

### Why It Matters
Prompts are the most impactful lever in an agent system. A small prompt change can radically alter behavior. If prompts are scattered across framework internals or config files, you cannot review, test, or roll back changes reliably.

### Compliance Check
- [ ] All prompts are stored as files in the repository (not inline strings in framework calls)
- [ ] Prompts are version-controlled with meaningful commit messages
- [ ] Prompt changes go through code review
- [ ] Prompts are parameterized (templates with variables, not hardcoded values)
- [ ] There is a prompt registry or index listing all active prompts

### Implementation Notes
- Store prompts in a dedicated directory (e.g., `prompts/` or `02_constraints/prompts/`).
- Use a template engine (Jinja2, Mustache, or simple f-strings) for parameterization.
- Include prompt version in logs for traceability.

---

## Factor 4: Own Your Context

### Description
You control exactly what goes into the LLM's context window. Context is curated, not accumulated. You decide what the LLM sees, in what order, and with what emphasis.

### Why It Matters
Context is the LLM's entire world. Irrelevant context wastes tokens and degrades performance. Missing context causes hallucination. Poorly ordered context buries important information. Context management is the single most impactful engineering discipline in agent systems.

### Compliance Check
- [ ] Context is explicitly constructed before each LLM call (not just an ever-growing message array)
- [ ] Irrelevant history is summarized or dropped, not appended verbatim
- [ ] Critical information is placed at the beginning and end of the context (primacy/recency effect)
- [ ] Context size is monitored and stays within budget
- [ ] There is a context assembly function that can be inspected and tested

### Implementation Notes
- Implement a `build_context()` function that assembles the prompt + relevant state + relevant history for each LLM call.
- Use summarization to compress old conversation turns.
- Tag context sections (system, user, tool results) so you can audit what the LLM saw.

---

## Factor 5: Compact LLM Calls into Tools

### Description
When you find yourself making multiple LLM calls to accomplish what could be a single deterministic function, extract that logic into a tool. LLM calls are expensive, slow, and non-deterministic. Tools are cheap, fast, and deterministic.

### Why It Matters
Every LLM call introduces latency, cost, and the possibility of error. If the same logic can be expressed as a Python function, a database query, or an API call, it should be. Reserve LLM calls for tasks that genuinely require reasoning, creativity, or natural language understanding.

### Compliance Check
- [ ] No LLM calls are used for tasks that could be deterministic functions
- [ ] Data retrieval is done by tools (DB queries, API calls), not by asking the LLM to recall
- [ ] Formatting and transformation are done by tools, not by LLM prompts
- [ ] Each LLM call has a clear purpose that requires LLM capabilities
- [ ] Tool results are passed back as structured data, not re-described in natural language

### Implementation Notes
- Audit your agent's LLM calls. For each one, ask: "Could a Python function do this?" If yes, extract it.
- Common candidates for tool extraction: date math, string formatting, data lookups, calculations, file I/O.

---

## Factor 6: Launch, Don't Land

### Description
Design agents to be launched (kicked off with a task and context) rather than landed on (expected to figure out what to do from a standing start). The harness prepares everything the agent needs, then hands off.

### Why It Matters
Agents that must discover their own task, gather their own context, and determine their own stopping criteria are fragile and unpredictable. Agents that receive a clear task, relevant context, and defined success criteria are reliable.

### Compliance Check
- [ ] Every agent invocation includes an explicit task description
- [ ] Relevant context is pre-gathered and injected (not discovered by the agent)
- [ ] Success criteria are defined before the agent starts
- [ ] The agent does not need to "figure out" what it should be doing
- [ ] Stopping conditions are explicit, not emergent

### Implementation Notes
- Think of agent invocation like a function call: it has inputs (task + context), a return type (expected output), and a contract (what success looks like).
- Pre-fetch data the agent will need rather than giving it search tools and hoping it finds the right data.

---

## Factor 7: Own Your Control Flow

### Description
Your code -- not the LLM, not the framework -- determines the order of operations. The LLM makes decisions within steps, but the step sequence is defined by your code.

### Why It Matters
LLM-driven control flow (e.g., ReAct loops where the LLM decides what to do next indefinitely) is unpredictable. It can loop, skip steps, or go off-track. Harness-driven control flow is deterministic and debuggable. You can add logging, metrics, and error handling at each step.

### Compliance Check
- [ ] The step sequence is defined in code, not determined by the LLM at runtime
- [ ] The LLM's role at each step is clearly scoped (decide X, generate Y, evaluate Z)
- [ ] Branching logic is explicit in code, informed by LLM output
- [ ] There are no unbounded LLM-driven loops
- [ ] Each step has a defined input, output, and failure mode

### Implementation Notes
- A common pattern: code defines the step sequence, the LLM fills in the "intelligence" at each step. Example: Step 1 (code: fetch data) -> Step 2 (LLM: analyze data) -> Step 3 (code: format output) -> Step 4 (LLM: generate summary).
- Use state machines or DAG definitions to make control flow explicit and visual.

---

## Factor 8: Own Your Working State

### Description
Working state (the data the agent accumulates and modifies during execution) lives in a data structure your code controls. It is not hidden inside a message array or framework abstraction.

### Why It Matters
If working state is spread across conversation messages, you cannot inspect it, persist it, or resume from it. Explicit state enables: checkpointing (save and resume), observability (inspect state at any point), debugging (reproduce issues by replaying from a state snapshot).

### Compliance Check
- [ ] Working state is stored in an explicit data structure (dict, dataclass, database row)
- [ ] State is updated by your code after each step, not implicitly by message accumulation
- [ ] State can be serialized, persisted, and restored
- [ ] State is logged at each step transition
- [ ] State schema is defined and documented

### Implementation Notes
- Define a `WorkflowState` class or TypedDict that captures everything the agent needs to know at any point.
- After each LLM call, extract relevant information from the response and update the state object.
- Never rely on the LLM's "memory" of previous messages as your source of truth.

---

## Factor 9: Tools Are Just Structured Output

### Description
When the LLM "calls a tool," it is producing structured data (a function name and arguments). Your code then executes the actual function. The LLM does not have the ability to execute anything.

### Why It Matters
This mental model is critical for safety, reliability, and testability. The LLM proposes actions; your code executes them. This means you can validate, filter, rate-limit, mock, and log every "tool call" before it actually happens.

### Compliance Check
- [ ] Tool calls are intercepted and validated by the harness before execution
- [ ] Tools can be mocked for testing without calling the LLM
- [ ] Dangerous tool calls (delete, write, send) require additional validation
- [ ] Tool call arguments are type-checked and sanitized
- [ ] Tool execution results are processed by the harness before being fed back to the LLM

### Implementation Notes
- Implement a `tool_executor` layer that receives the LLM's structured output, validates it, executes the function, and formats the result.
- Add a "dry run" mode where tool calls are logged but not executed.
- Use JSON Schema to define tool interfaces and validate arguments.

---

## Factor 10: Don't Be Precious About Agent Boundaries

### Description
Agent boundaries should be drawn by engineering pragmatism, not philosophical purity. Merge agents when the communication overhead exceeds the benefit. Split agents when a single agent's scope exceeds a single context window or when parallel execution would help.

### Why It Matters
Over-decomposed systems (too many tiny agents) suffer from coordination overhead and information loss. Under-decomposed systems (one giant agent) suffer from context window limits and tangled prompts. The right granularity depends on the task, and it changes as the system evolves.

### Compliance Check
- [ ] Agent boundaries are justified by concrete engineering reasons (context limits, parallelism, specialization)
- [ ] There is no agent that exists solely for "architectural elegance"
- [ ] Inter-agent communication cost has been evaluated
- [ ] Agent boundaries have been revisited in the last 3 months
- [ ] Merging or splitting agents is considered during retrospectives

### Implementation Notes
- Start with fewer, larger agents. Split when you hit a concrete problem (context too large, prompt too complex, steps that could run in parallel).
- When splitting, ask: "Does the overhead of passing context between these agents exceed the benefit of separation?"

---

## Factor 11: Treat Your Agent Framework Like a Database Driver

### Description
Your agent framework (LangChain, CrewAI, raw API calls) is an implementation detail, not the architecture. You should be able to swap it without rewriting your business logic.

### Why It Matters
The agent tooling ecosystem is evolving rapidly. Frameworks that are popular today may be obsolete tomorrow. If your business logic is entangled with framework internals, you are locked in. Treating the framework as a thin interface layer gives you freedom to evolve.

### Compliance Check
- [ ] Business logic is separated from framework-specific code
- [ ] LLM calls go through an abstraction layer (not direct framework calls in business logic)
- [ ] Prompts are stored independently of the framework
- [ ] Switching from one LLM provider to another requires changes in only one layer
- [ ] No framework-specific types leak into the core domain

### Implementation Notes
- Create a thin wrapper module (e.g., `llm_client.py`) that exposes `complete(prompt, tools) -> response`. This is the only place that imports the framework.
- Your workflow code calls `llm_client.complete()`, not `langchain.ChatOpenAI().invoke()`.
- This also makes testing easy: mock the wrapper, not the framework.

---

## Factor 12: Make Agents a DAG

### Description
Every agent workflow can be represented as a directed acyclic graph (DAG). Nodes are steps (LLM calls, tool executions, validation checks). Edges are transitions (conditional or unconditional). There are no cycles (use iteration with explicit counters instead).

### Why It Matters
DAG representation gives you: visualization (draw the workflow), validation (check for unreachable nodes), parallelization (independent nodes can run concurrently), debugging (trace the exact path through the graph), and monitoring (instrument each node independently).

### Compliance Check
- [ ] The workflow can be drawn as a DAG
- [ ] There are no uncounted cycles (all loops have explicit iteration limits)
- [ ] Independent steps are identified and can be parallelized
- [ ] Each node in the DAG has a single, well-defined responsibility
- [ ] The DAG is documented (either as a diagram or code comments)

### Implementation Notes
- Even if you don't use a formal DAG framework, draw your workflow as a DAG on paper or in ASCII art. If you cannot, the design is too tangled.
- Use topological ordering to determine which steps can run in parallel.
- Implement iteration as a node that increments a counter and transitions back to an earlier node, with a guard condition on the counter.

---

## Usage

### Initial Audit
Walk through all 12 factors for your agent system. Check every box you can honestly check. The unchecked boxes are your improvement backlog.

### Design Review
Before building a new agent, review this checklist to ensure the design addresses all 12 factors.

### Retrospective
After an incident or a sprint, revisit the checklist. Incidents often trace back to a violated factor.

### Scoring
- **10-12 checked**: Production-ready harness engineering.
- **7-9 checked**: Solid foundation, address gaps before scaling.
- **4-6 checked**: Significant risk. Prioritize compliance before adding features.
- **0-3 checked**: Prototype stage. Do not deploy to production.
