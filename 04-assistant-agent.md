# Part 4: AssistantAgent - The Core AI Agent

## What is AssistantAgent?

**AssistantAgent** is the primary AI-powered agent in AutoGen. It:
- Processes tasks by calling an LLM (via model_client)
- Calls tools based on LLM decisions
- Iterates: LLM call → tool execution → LLM reflection → repeat
- Maintains conversation history
- Can hand off to other agents

```
Task
  ↓
AssistantAgent processes
  ├── Sees available tools
  ├── Calls LLM (via model_client)
  ├── LLM decides: tool call or respond?
  ├── If tool: Execute and feed result back to LLM
  ├── If respond: Return to caller
  └── Repeat up to max_tool_iterations
```

---

## Full Constructor Signature

```python
from autogen_agentchat.agents import AssistantAgent

agent = AssistantAgent(
    # Required
    name: str,
    model_client: ChatCompletionClient,
    
    # Optional - System & Behavior
    system_message: str = "You are a helpful assistant.",
    tools: list | None = None,
    reflect_on_tool_use: bool = False,
    max_tool_iterations: int = 10,
    
    # Optional - Memory & Context
    model_context: ModelContext | None = None,
    memory: Memory | None = None,
    
    # Optional - Advanced
    handoffs: list[str] | None = None,
    workbench: WorkbenchType | None = None,
)
```

---

## Constructor Parameters: Complete Guide

### Required Parameters

#### `name: str`
The agent's identifier.

**Purpose**: Used in messages to identify who is speaking.

```python
agent = AssistantAgent(
    name="researcher",  # Must be unique in team
    model_client=model_client
)
```

**Typical names**:
- `"researcher"` - Gathers information
- `"analyzer"` - Analyzes data  
- `"writer"` - Writes content
- `"reviewer"` - Reviews work
- `"coordinator"` - Orchestrates

**If omitted**: Will raise ValueError

---

#### `model_client: ChatCompletionClient`
The LLM interface for this agent.

**Purpose**: Specifies which LLM to use and how to call it.

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient

model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY")
)

agent = AssistantAgent(
    name="researcher",
    model_client=model_client
)
```

**Why separate from agent?**
- Same agent can use different LLMs
- Easy to swap models
- Easy to test with mock clients

**If omitted**: Will raise ValueError

---

### Optional Parameters - System & Behavior

#### `system_message: str`
The system prompt that defines agent's role.

**Default**: `"You are a helpful assistant."`

**Purpose**: Controls agent behavior and persona.

**Examples**:

```python
# Research agent
agent = AssistantAgent(
    name="researcher",
    model_client=model_client,
    system_message="""You are an expert researcher. Your role is to:
1. Search for reliable information sources
2. Evaluate source credibility
3. Synthesize findings
4. Always cite sources

Be thorough and critical. Question questionable claims."""
)

# Code reviewer
agent = AssistantAgent(
    name="reviewer",
    model_client=model_client,
    system_message="""You are a senior code reviewer. 
- Find bugs and vulnerabilities
- Check for performance issues
- Suggest improvements
- Be constructive but firm
- Explain your reasoning"""
)

# Data analyst
agent = AssistantAgent(
    name="analyst",
    model_client=model_client,
    system_message="""You are a data analyst.
- Clean and validate data
- Find patterns and outliers
- Provide statistical insights
- Use visualizations when helpful"""
)
```

**Important**: System message is sent to LLM, not executed locally.

**Common mistake**: Including hard instructions that require local computation

```python
# WRONG - Can't count locally in system message
system_message="Count all tool calls and report the total"

# RIGHT - System message defines role
system_message="You are a helpful assistant"
```

---

#### `tools: list | None`
Functions the agent can call.

**Default**: `None` (no tools)

**Purpose**: Extends agent capabilities.

```python
def web_search(query: str) -> str:
    """Search the web"""
    ...

def code_executor(code: str) -> str:
    """Execute Python code"""
    ...

agent = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[web_search, code_executor]
)
```

**How tools work**:
1. Tools passed to agent
2. Agent sends tool descriptions to LLM
3. LLM decides if tool call needed
4. LLM generates tool_call JSON
5. Agent executes tool
6. Results fed back to LLM
7. LLM continues or returns response

**Important**: Tools are Python functions with type hints

```python
from typing import Annotated

def calculate_sum(
    numbers: Annotated[list[int], "List of numbers to sum"]
) -> str:
    """Calculate sum of numbers"""
    return str(sum(numbers))

agent = AssistantAgent(
    tools=[calculate_sum]
)
```

**If omitted**: Agent cannot call any tools, only generate text responses.

---

#### `reflect_on_tool_use: bool`
Whether agent should verify tool results.

**Default**: `False`

**Purpose**: Adds reflection step after tool calls for better quality.

**What it does**:

```
WITHOUT reflect_on_tool_use (default):
Task → LLM → "Call tool X" → Execute tool → LLM → Response

WITH reflect_on_tool_use:
Task → LLM → "Call tool X" → Execute tool → 
    LLM: "Let me verify this result" → 
    Verification check → Final response
```

**Example**:

```python
# Without reflection
agent = AssistantAgent(
    name="analyst",
    model_client=model_client,
    tools=[data_processor],
    reflect_on_tool_use=False  # Default
)

# With reflection
agent = AssistantAgent(
    name="analyst",
    model_client=model_client,
    tools=[data_processor],
    reflect_on_tool_use=True  # Add verification step
)
```

**When to use**:
- ✅ When tool results must be accurate (medical, financial)
- ✅ When tools might fail or return unexpected results
- ✅ When you need quality assurance
- ❌ When speed is critical (adds LLM call)
- ❌ When tool results are always correct

**Token cost**: Adds 1-2 extra LLM calls per tool use.

---

#### `max_tool_iterations: int`
Maximum tool calls before stopping.

**Default**: `10`

**Purpose**: Prevents infinite tool-calling loops.

**Behavior**:

```
max_tool_iterations = 3 example:

Iteration 1: LLM → "Call tool" → Tool result
Iteration 2: LLM → "Call tool" → Tool result  
Iteration 3: LLM → "Call tool" → Tool result
Iteration 4: Would exceed max → Stop, return result
```

**Example**:

```python
# For simple tasks (few tool calls expected)
agent = AssistantAgent(
    name="simple",
    tools=[web_search],
    max_tool_iterations=2
)

# For complex reasoning (many steps)
agent = AssistantAgent(
    name="complex",
    tools=[search, analyze, compute, verify],
    max_tool_iterations=15
)
```

**Important distinctions**:
- `max_tool_iterations` ≠ `max_messages` (total LLM calls)
- `max_tool_iterations` = specifically tool calls
- After reaching max, agent returns best response

**If omitted**: Defaults to 10 (usually sufficient).

**Common mistake**: Setting too low

```python
# WRONG - Will stop before finishing complex task
agent = AssistantAgent(
    max_tool_iterations=1  # Only 1 tool call allowed
)

# RIGHT - Allow enough for task complexity
agent = AssistantAgent(
    max_tool_iterations=5  # For typical tasks
)
```

---

### Optional Parameters - Memory & Context

#### `model_context: ModelContext | None`
Manages what LLM sees (conversation history window).

**Default**: `None` (uses default context management)

**Purpose**: Controls token usage by selecting what to send to LLM.

**Without model_context**:
```
Every LLM call includes:
- Entire conversation history
- All past messages
- Grows with each call
- Eventually hits token limits
```

**With model_context**:
```
LLM only sees:
- Recent relevant messages
- Summary of past if needed
- Pruned irrelevant information
- Keeps within token budget
```

**Example**:

```python
# Complex conversation with many messages
agent = AssistantAgent(
    name="researcher",
    model_client=model_client,
    model_context=SomeContextManager(
        max_window=10,  # Last 10 messages
    )
)
```

**Version-dependent**: Specific context implementations may vary.

**If omitted**: Uses AutoGen's default context strategy.

---

#### `memory: Memory | None`
Long-term memory system.

**Default**: `None` (no persistent memory)

**Purpose**: Remember useful information across multiple tasks.

**Without memory**:
```
Task 1 → Agent processes → Response → Forgotten

Task 2 → Agent has no context from Task 1
```

**With memory**:
```
Task 1 → Agent processes → Important facts stored in memory

Task 2 → Agent retrieves relevant facts from memory
       → Uses them in reasoning → Better response
```

**Example**:

```python
# Agent with memory
agent = AssistantAgent(
    name="researcher",
    model_client=model_client,
    memory=SomeMemorySystem()  # Stores facts
)

# Task 1
await agent.run(task="What's the population of France?")

# Task 2 - can retrieve population fact from Task 1
await agent.run(task="What's the population density of France?")
# Memory helps with second task
```

**If omitted**: Agent starts fresh for each task (no learning).

---

### Optional Parameters - Advanced

#### `handoffs: list[str] | None`
Which agents this agent can transfer to.

**Default**: `None` (no handoffs)

**Purpose**: Enables agent-to-agent communication and delegation.

**Example**:

```python
# Researcher can hand off to writer
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    handoffs=["writer"]  # Can transfer to writer
)

# Writer can hand off to reviewer
writer = AssistantAgent(
    name="writer",
    model_client=model_client,
    handoffs=["reviewer"]  # Can transfer to reviewer
)

# Create team with these agents
team = RoundRobinGroupChat(
    participants=[researcher, writer, reviewer]
)
```

**Handoff message**:
```python
from autogen_agentchat.messages import HandoffMessage

# Researcher decides to hand off
handoff = HandoffMessage(
    to="writer",  # Must be in handoffs list
    content="Here's what I researched..."
)
```

**Important**: Declaring handoffs doesn't automatically implement routing logic. The team orchestration handles routing.

**If omitted**: Agent cannot hand off (stays until done).

---

#### `workbench: WorkbenchType | None`
Experimental tool execution environment.

**Default**: `None`

**Purpose**: Sandbox for tool execution (version-dependent).

**Status**: Experimental/Advanced feature

**If omitted**: Tools execute in default environment.

---

## Agent Execution Flow: Detailed

```
┌────────────────────────────────────┐
│ Task Received                      │
│ "Research Python 3.13 features"    │
└────────────┬───────────────────────┘
             │
┌────────────▼───────────────────────┐
│ Agent.on_messages() Called          │
│                                    │
│ • Prepare context                  │
│ • Include system_message           │
│ • Include conversation history     │
│ • Include available tools (schema) │
└────────────┬───────────────────────┘
             │
┌────────────▼───────────────────────┐
│ Call model_client.create()          │
│                                    │
│ Send to LLM:                       │
│ {                                  │
│   "model": "gpt-4",               │
│   "messages": [                   │
│     {"role": "system", ...},     │
│     {"role": "user", ...}        │
│   ],                              │
│   "tools": [                      │
│     web_search schema,            │
│     ...                           │
│   ]                               │
│ }                                  │
└────────────┬───────────────────────┘
             │
┌────────────▼───────────────────────┐
│ LLM Processes Request               │
│                                    │
│ Sees:                              │
│ • Task description                 │
│ • System message (role)            │
│ • Previous messages (history)      │
│ • Available tools                  │
│                                    │
│ Decides:                           │
│ Option A: "I need web_search"      │
│ Option B: "I can answer directly"  │
│ Option C: "I need more context"    │
└────────────┬───────────────────────┘
             │
    ┌────────┴──────────┐
    │                   │
┌───▼────────────┐  ┌──▼────────────────┐
│ Option A       │  │ Option B or C      │
│ Tool call      │  │ Text response      │
└───┬────────────┘  │                    │
    │               │ Return to caller   │
┌───▼────────────────────────────────┐  │
│ Execute Tool                        │  │
│                                    │  │
│ tool_result = web_search(         │  │
│    "Python 3.13 features"        │  │
│ )                                  │  │
└───┬────────────────────────────────┘  │
    │                                   │
┌───▼────────────────────────────────┐  │
│ Add to Context                      │  │
│                                    │  │
│ • Tool call request event         │  │
│ • Tool execution event            │  │
│ • Tool result (web_search output) │  │
└───┬────────────────────────────────┘  │
    │                                   │
┌───▼────────────────────────────────┐  │
│ Check: max_tool_iterations?        │  │
│                                    │  │
│ Count < max? → Continue           │  │
│ Count >= max? → Stop, return       │  │
└───┬────────────────────────────────┘  │
    │ (if continue)                     │
┌───▼────────────────────────────────┐  │
│ reflect_on_tool_use?               │  │
│                                    │  │
│ If True:                           │  │
│ • Call LLM again to verify result  │  │
│ • LLM validates tool output        │  │
│ • Adds verification step           │  │
│                                    │  │
│ If False:                          │  │
│ • Skip verification               │  │
│ • Go to next LLM call             │  │
└───┬────────────────────────────────┘  │
    │                                   │
┌───▼────────────────────────────────┐  │
│ Call LLM Again                     │  │
│                                    │  │
│ LLM sees:                          │  │
│ • Original task                    │  │
│ • Tool result                      │  │
│ • Previous messages                │  │
│                                    │  │
│ Decides: Need more tools? Or done? │  │
│                                    │  │
│ Loop back or exit                 │  │
└───┴────────────────────────────────┘  │
                                        │
                 ┌──────────────────────┘
                 │
            ┌────▼────────────────────┐
            │ Final Response           │
            │                        │
            │ Text response to caller │
            └────────────────────────┘
```

---

## Practical Examples

### Example 1: Simple Research Agent

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient
import os

def web_search(query: str) -> str:
    """Search the web for information"""
    # Implementation details omitted
    return f"Search results for: {query}"

# Setup
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY")
)

# Create agent
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    system_message="You are a researcher. Search for information and provide thorough answers.",
    tools=[web_search],
    max_tool_iterations=5
)

# Use agent
async def main():
    result = await researcher.run(
        task="Research recent developments in quantum computing"
    )
    print(result.messages[-1].content)

asyncio.run(main())
```

### Example 2: Code Review Agent with Reflection

```python
from autogen_agentchat.agents import AssistantAgent

def run_tests(code: str) -> str:
    """Run tests on code"""
    ...

def analyze_complexity(code: str) -> str:
    """Analyze code complexity"""
    ...

# Code reviewer with reflection
reviewer = AssistantAgent(
    name="code_reviewer",
    model_client=model_client,
    system_message="""You are a senior code reviewer.
    - Find bugs and vulnerabilities
    - Check code quality
    - Suggest improvements
    - Always verify your findings""",
    tools=[run_tests, analyze_complexity],
    reflect_on_tool_use=True,  # Verify findings
    max_tool_iterations=8
)

# Will:
# 1. Analyze code (tool call)
# 2. Run tests (tool call)
# 3. Reflect on results (extra LLM call)
# 4. Provide verified findings
```

### Example 3: Data Analyst with Handoff

```python
# Data analyst that can hand off to writer
analyst = AssistantAgent(
    name="analyst",
    model_client=model_client,
    system_message="Analyze data and provide insights",
    tools=[process_data, calculate_stats],
    handoffs=["writer"],  # Can hand off to writer
    max_tool_iterations=6
)

# Writer that can hand off to reviewer
writer = AssistantAgent(
    name="writer",
    model_client=model_client,
    system_message="Write clear reports based on analysis",
    handoffs=["reviewer"]
)

# Reviewer
reviewer = AssistantAgent(
    name="reviewer",
    model_client=model_client,
    system_message="Review and approve reports"
)
```

---

## Common Mistakes & Fixes

### Mistake 1: No Tools Provided

```python
# WRONG - Agent can't do anything but talk
agent = AssistantAgent(
    name="agent",
    model_client=model_client
    # No tools!
)

# Can only generate text, can't execute anything

# FIX
agent = AssistantAgent(
    name="agent",
    model_client=model_client,
    tools=[web_search, calculate, retrieve_data]
)
```

### Mistake 2: max_tool_iterations Too Low

```python
# WRONG - Stops too early
agent = AssistantAgent(
    max_tool_iterations=1  # Only 1 tool call!
)

# Task requires 3 tool calls but stops at 1

# FIX
agent = AssistantAgent(
    max_tool_iterations=10  # Enough for most tasks
)
```

### Mistake 3: Unclear System Message

```python
# WRONG - Too vague
system_message="You are helpful"

# Agent doesn't know what to do

# FIX - Be specific
system_message="""You are a research analyst specializing in technology.
- Use web_search to find credible sources
- Synthesize findings into coherent summaries
- Always cite sources
- Be critical of unusual claims"""
```

### Mistake 4: Not Passing model_client

```python
# WRONG - Will crash
agent = AssistantAgent(
    name="agent"
    # Missing model_client!
)

# FIX
agent = AssistantAgent(
    name="agent",
    model_client=OpenAIChatCompletionClient(model="gpt-4")
)
```

---

## What You Get Back: Understanding TaskResult

When you call `await agent.run()`, you get a **TaskResult** object containing:
- `messages` - All conversation messages
- `stop_reason` - Why execution stopped
- `metadata` - Execution details (tokens used, timing, etc.)

**To understand exactly what's in these structures, see [31-runtime-outputs-objects.md](31-runtime-outputs-objects.md)**

This is critical for:
- Extracting the agent's response
- Understanding why execution stopped
- Analyzing tool calls and results
- Tracking token usage

---

## Next Steps

Now you understand:
- ✅ AssistantAgent constructor
- ✅ All parameters and their purposes
- ✅ Execution flow
- ✅ Common mistakes

Move to:
- **[09-tools.md](09-tools.md)** to master tool creation
- **[10-single-agent-loop.md](10-single-agent-loop.md)** to understand complete execution
- **[31-runtime-outputs-objects.md](31-runtime-outputs-objects.md)** to understand what you receive ⭐ **NEW**
- **[11-teams.md](11-teams.md)** to orchestrate multiple agents

---

## References

- [AutoGen AssistantAgent Documentation](https://microsoft.github.io/autogen/)
- [AutoGen Source Code - agents/assistant_agent.py](https://github.com/microsoft/autogen)

---

## Version Notes

- **Current Version**: AutoGen 0.2.x
- **Parameters**: Subject to minor changes between versions
- **If uncertain**: Verify against `agent.py` source code for exact defaults
