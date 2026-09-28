# Part 31: Runtime Outputs, Return Types & Object Anatomy

## Overview

Understanding AutoGen APIs requires knowing not just *what* methods do, but *what objects you receive at runtime*.

This document explains:
- Exact Python objects returned by every important AutoGen API
- Field names, types, and nested structures
- How to inspect and access values
- Real runtime examples
- Common pitfalls when working with return values

---

## Principle: API ≠ Return Value

Many developers understand:
```python
result = await agent.run(task="...")
```

But then don't know:
- What is `result`? (Type name? Data class?)
- What attributes does it have?
- What types are those attributes?
- What nested structures exist?
- How do I safely extract values?

This section fills that gap.

---

## Foundation: How to Inspect Objects at Runtime

Before diving into specific objects, here's how to inspect any AutoGen object:

```python
# Get the type
print(type(result))
# Output: <class 'autogen_agentchat.task.TaskResult'>

# Get the class name
print(result.__class__.__name__)
# Output: TaskResult

# List all attributes
print(dir(result))
# Output: ['messages', 'stop_reason', 'metadata', ...]

# Get attributes and their types
import inspect
for name, value in vars(result).items():
    print(f"{name}: {type(value).__name__} = {value}")

# If it's a dataclass or Pydantic model
if hasattr(result, '__dataclass_fields__'):
    print(result.__dataclass_fields__)
elif hasattr(result, 'model_fields'):
    print(result.model_fields)

# Pretty print
import json
if hasattr(result, 'model_dump'):
    print(json.dumps(result.model_dump(), indent=2, default=str))
elif hasattr(result, 'to_dict'):
    print(json.dumps(result.to_dict(), indent=2, default=str))
```

---

## 1. TaskResult Object

### What it is
Returned by `agent.run()` and `team.run()`. Contains complete execution results.

### Call Example
```python
result = await agent.run(task="Research Python 3.13")
```

### Return Type
```python
type(result)
# → autogen_agentchat.task.TaskResult
```

### Object Structure

```python
TaskResult
├── messages: list[Message]
│   ├── [0]: TextMessage
│   │   ├── content: str
│   │   ├── source: str
│   │   └── models_usage: ModelUsage | None
│   ├── [1]: ToolCallMessage
│   ├── [2]: ToolExecutionSummaryMessage
│   └── [n]: ...
│
├── stop_reason: str
│   └── Values: "response", "max_tool_iterations", "max_turns", "cancelled", ...
│
└── metadata: dict | None
    ├── "execution_time": float (seconds)
    ├── "token_usage": dict (if available)
    ├── "error": str (if failed)
    └── ... (provider-specific)
```

### Field Documentation

#### `messages: list[Message]`

**Type**: Python list of Message objects

**What it contains**: Complete conversation from start to finish

**Structure**:
- Each message is a Message subclass (TextMessage, HandoffMessage, etc.)
- Order matters: chronological execution order
- Can be empty if execution failed

**How to access**:
```python
# Get all messages
all_messages = result.messages

# Get last message
final_message = result.messages[-1]
print(final_message.content)

# Get first message
first_message = result.messages[0]

# Filter messages by type
text_messages = [m for m in result.messages if isinstance(m, TextMessage)]

# Count messages
num_messages = len(result.messages)

# Iterate with content
for i, msg in enumerate(result.messages):
    print(f"Message {i}: {msg.source} → {msg.content[:100]}")
```

**Runtime Example**:
```python
result = await agent.run(task="Search for Python news")

print(f"Total messages: {len(result.messages)}")
# Output: Total messages: 4

for msg in result.messages:
    print(f"[{msg.source}] {msg.content[:80]}...")
# Output:
# [researcher] I'll search for the latest Python news for you.
# [researcher] Searching...
# [web_search] [tool output] Found 3 news articles about Python...
# [researcher] Here's what I found about Python developments:
```

---

#### `stop_reason: str`

**Type**: String enum (but not a Python Enum, just a string)

**Possible values**:
- `"response"` - Agent responded with text (not tool call)
- `"max_tool_iterations"` - Reached max tool call limit
- `"max_turns"` - Reached max conversation turns
- `"cancelled"` - Execution was cancelled
- `"termination_condition"` - Custom termination condition met
- `"error"` - Execution failed

**How to use**:
```python
if result.stop_reason == "response":
    print("✅ Completed normally")
elif result.stop_reason == "max_tool_iterations":
    print("⚠️  Stopped due to tool iteration limit")
elif result.stop_reason == "max_turns":
    print("⚠️  Stopped due to turn limit")
elif result.stop_reason == "cancelled":
    print("❌ Execution was cancelled")
else:
    print(f"⚠️  Unknown reason: {result.stop_reason}")

# Check reason programmatically
reasons_needing_attention = {"max_tool_iterations", "max_turns", "error"}
if result.stop_reason in reasons_needing_attention:
    # Handle incomplete result
    pass
```

**Runtime Example**:
```python
result = await agent.run(task="Complex analysis")

if result.stop_reason == "response":
    # Use result
    final = result.messages[-1].content
else:
    print(f"Task incomplete: {result.stop_reason}")
    # Log for review
```

---

#### `metadata: dict | None`

**Type**: Optional Python dictionary

**What it contains**: Execution metadata (varies by implementation)

**Typical keys**:
```python
{
    "execution_time": 12.34,  # seconds
    "token_usage": {
        "input_tokens": 2048,
        "output_tokens": 1024,
        "total_tokens": 3072
    },
    "model_calls": 3,  # How many LLM calls
    "tool_calls": 2,   # How many tools executed
    "error": None,     # Error message if failed
    "agents_used": ["researcher", "analyzer"],
    "team_type": "RoundRobinGroupChat"
}
```

**How to access**:
```python
if result.metadata:
    # Safe access with defaults
    exec_time = result.metadata.get("execution_time", 0)
    total_tokens = result.metadata.get("token_usage", {}).get("total_tokens", 0)
    
    print(f"Execution time: {exec_time:.2f}s")
    print(f"Total tokens: {total_tokens}")
else:
    print("No metadata available")

# Check for errors
if result.metadata and "error" in result.metadata:
    print(f"Error occurred: {result.metadata['error']}")
```

**Version Note**: Token usage fields are model_client and provider-dependent. Not all providers populate this.

---

## 2. TextMessage Object

### What it is
Represents a text message from an agent.

### Structure
```python
TextMessage
├── content: str
├── source: str
├── models_usage: ModelUsage | None
└── [internal fields]
```

### Field Documentation

#### `content: str`
The actual message text.

```python
message = result.messages[0]
text = message.content  # String with agent's response

# Find specific content
if "error" in message.content.lower():
    print("Message contains error reference")

# Extract first N characters
summary = message.content[:200]
```

#### `source: str`
Which agent sent this message.

```python
message = result.messages[0]
sender = message.source  # e.g., "researcher"

# Filter messages from specific agent
researcher_messages = [m for m in result.messages if m.source == "researcher"]

# Track which agent speaks
speakers = set(m.source for m in result.messages)
print(f"Agents that spoke: {speakers}")
```

#### `models_usage: ModelUsage | None`
Token usage for this specific message (if available).

```python
message = result.messages[0]

if message.models_usage:
    tokens_in = message.models_usage.prompt_tokens
    tokens_out = message.models_usage.completion_tokens
    total = tokens_in + tokens_out
    print(f"This message used {total} tokens")
else:
    print("Token usage not available for this message")
```

### Runtime Example

```python
result = await agent.run(task="Search for Python")

# Iterate through messages
for message in result.messages:
    print(f"\n--- {message.source.upper()} ---")
    print(f"Message: {message.content}")
    
    if message.models_usage:
        print(f"Tokens: {message.models_usage.prompt_tokens + message.models_usage.completion_tokens}")

# Output example:
# --- RESEARCHER ---
# Message: I'll search for information about Python.
# Tokens: 150

# --- RESEARCHER ---
# Message: Based on my search: [results here]
# Tokens: 2048
```

---

## 3. HandoffMessage Object

### What it is
Represents transfer of control from one agent to another.

### Structure
```python
HandoffMessage
├── to: str              # Target agent name
├── from_agent: str      # Source agent name  
├── content: str         # Transfer context
└── models_usage: ModelUsage | None
```

### Field Documentation

```python
# Find handoff messages
handoffs = [m for m in result.messages if isinstance(m, HandoffMessage)]

for handoff in handoffs:
    print(f"{handoff.from_agent} → {handoff.to}")
    print(f"Context: {handoff.content}")

# Track handoff chain
handoff_chain = []
for msg in result.messages:
    if isinstance(msg, HandoffMessage):
        handoff_chain.append((msg.from_agent, msg.to))

print(f"Agent chain: {' → '.join([h[0] for h in handoff_chain] + [handoff_chain[-1][1]])}")
# Output: Agent chain: researcher → analyzer → writer
```

---

## 4. ToolCallSummaryMessage Object

### What it is
Represents tool calls made by an agent.

### Structure
```python
ToolCallSummaryMessage
├── tool_calls: list[ToolCall]
│   └── [0]: ToolCall
│       ├── id: str
│       ├── function_name: str
│       ├── arguments: dict
│       └── result: str
├── source: str
└── models_usage: ModelUsage | None
```

### How to access tool calls

```python
# Find tool call messages
from autogen_agentchat.messages import ToolCallSummaryMessage

tool_messages = [m for m in result.messages if isinstance(m, ToolCallSummaryMessage)]

for tool_msg in tool_messages:
    print(f"Agent {tool_msg.source} made {len(tool_msg.tool_calls)} tool calls:")
    
    for tool_call in tool_msg.tool_calls:
        print(f"  - {tool_call.function_name}({tool_call.arguments})")
        print(f"    Result: {tool_call.result[:100]}...")

# Count total tool calls
total_tool_calls = sum(len(m.tool_calls) for m in tool_messages)
print(f"Total tool calls: {total_tool_calls}")

# Find specific tool usage
web_searches = []
for msg in tool_messages:
    for call in msg.tool_calls:
        if call.function_name == "web_search":
            web_searches.append(call)

print(f"Web searches made: {len(web_searches)}")
```

---

## 5. ModelUsage Object

### What it is
Token usage information (if available).

### Structure
```python
ModelUsage
├── prompt_tokens: int
├── completion_tokens: int
└── total_tokens: int (optional)
```

### How to use

```python
def calculate_cost(usage, model="gpt-4"):
    # GPT-4 pricing
    input_cost = (usage.prompt_tokens / 1000) * 0.03
    output_cost = (usage.completion_tokens / 1000) * 0.06
    return input_cost + output_cost

# Calculate cost of entire execution
total_input = 0
total_output = 0

for msg in result.messages:
    if msg.models_usage:
        total_input += msg.models_usage.prompt_tokens
        total_output += msg.models_usage.completion_tokens

total_cost = (total_input / 1000 * 0.03) + (total_output / 1000 * 0.06)
print(f"Estimated cost: ${total_cost:.4f}")
print(f"Tokens: {total_input} input + {total_output} output = {total_input + total_output} total")
```

**Version Note**: Token counts depend on model client implementation. Some clients don't provide this information.

---

## 6. Agent.run_stream() Output

### What it is
An async generator that yields events during execution.

### Call Example
```python
async for event in agent.run_stream(task="..."):
    print(event)
```

### Event Types
```python
# Events can be various types:
# - TextMessage
# - ToolCallSummaryMessage
# - HandoffMessage
# - TaskResult (final event)
# - Internal events (depends on implementation)

async for event in agent.run_stream(task="Research"):
    if isinstance(event, TaskResult):
        print("Execution complete")
        break
    elif isinstance(event, ToolCallSummaryMessage):
        print(f"Tool called: {event.tool_calls[0].function_name}")
    elif isinstance(event, TextMessage):
        print(f"Agent said: {event.content[:100]}")
    else:
        print(f"Event: {type(event).__name__}")
```

### Access pattern

```python
results = []
async for event in agent.run_stream(task="..."):
    results.append(event)
    
    # Real-time handling
    if isinstance(event, ToolCallSummaryMessage):
        # Handle tool execution
        print(f"Tool: {event.tool_calls[0].function_name}")

# Final result
final_result = results[-1]  # Should be TaskResult
if isinstance(final_result, TaskResult):
    print(f"Stop reason: {final_result.stop_reason}")
```

---

## 7. Team.run() Output

### What it is
Same as Agent.run() - returns TaskResult.

### Structure
```python
result = await team.run(task="...")

# Same structure as agent
result.messages      # All messages from all agents
result.stop_reason   # Why execution stopped
result.metadata      # Execution metadata
```

### Key difference from single agent

```python
# Single agent: messages from one agent
agent_result = await agent.run(task="...")
# result.messages = [msg1 (agent), msg2 (agent), ...]

# Team: messages from multiple agents
team_result = await team.run(task="...")
# result.messages = [
#     msg1 (researcher),
#     msg2 (analyzer),
#     msg3 (researcher again),
#     msg4 (writer),
#     ...
# ]

# Track which agent at each step
for msg in team_result.messages:
    print(f"Step {len(messages)}: {msg.source}")
```

---

## 8. Nested Structure Example: Complete Inspection

```python
# Given a complete result, inspect everything
result = await agent.run(task="Search for Python")

def inspect_task_result(result):
    """Complete inspection of TaskResult"""
    
    print("=" * 60)
    print("TASKRESULT INSPECTION")
    print("=" * 60)
    
    # Top level
    print(f"\nType: {type(result)}")
    print(f"Stop reason: {result.stop_reason}")
    print(f"Has metadata: {result.metadata is not None}")
    
    # Messages summary
    print(f"\nMessages ({len(result.messages)} total):")
    for i, msg in enumerate(result.messages):
        msg_type = type(msg).__name__
        source = getattr(msg, 'source', 'unknown')
        content_preview = getattr(msg, 'content', '')[:50]
        print(f"  [{i}] {msg_type} from {source}: {content_preview}...")
    
    # Token usage
    print(f"\nToken Usage:")
    total_in = 0
    total_out = 0
    for msg in result.messages:
        if hasattr(msg, 'models_usage') and msg.models_usage:
            total_in += msg.models_usage.prompt_tokens
            total_out += msg.models_usage.completion_tokens
    print(f"  Input: {total_in}")
    print(f"  Output: {total_out}")
    print(f"  Total: {total_in + total_out}")
    
    # Tool calls
    print(f"\nTool Calls:")
    from autogen_agentchat.messages import ToolCallSummaryMessage
    total_tools = 0
    for msg in result.messages:
        if isinstance(msg, ToolCallSummaryMessage):
            total_tools += len(msg.tool_calls)
            for call in msg.tool_calls:
                print(f"  - {call.function_name}()")
    print(f"  Total: {total_tools}")
    
    # Metadata
    if result.metadata:
        print(f"\nMetadata:")
        for key, value in result.metadata.items():
            print(f"  {key}: {value}")

inspect_task_result(result)
```

---

## 9. Accessing Specific Information

### Get final response
```python
result = await agent.run(...)

# Last message from agent
final_response = result.messages[-1].content
print(final_response)
```

### Get all tool calls in order
```python
from autogen_agentchat.messages import ToolCallSummaryMessage

tool_calls = []
for msg in result.messages:
    if isinstance(msg, ToolCallSummaryMessage):
        for call in msg.tool_calls:
            tool_calls.append({
                "agent": msg.source,
                "function": call.function_name,
                "arguments": call.arguments,
                "result": call.result
            })

# Display tool execution timeline
for i, call in enumerate(tool_calls, 1):
    print(f"{i}. {call['agent']} calls {call['function']}")
    print(f"   Args: {call['arguments']}")
    print(f"   Result: {call['result'][:100]}...")
```

### Calculate total execution time
```python
if result.metadata and "execution_time" in result.metadata:
    exec_time = result.metadata["execution_time"]
    print(f"Execution took {exec_time:.2f} seconds")
```

### Check if task completed successfully
```python
def task_succeeded(result):
    """Check if task completed without issues"""
    if result.stop_reason == "response":
        return True
    if result.stop_reason == "max_tool_iterations":
        return False  # Incomplete
    if result.stop_reason == "error":
        return False  # Failed
    if result.metadata and "error" in result.metadata:
        return False  # Error occurred
    return None  # Unknown

success = task_succeeded(result)
if success:
    print("✅ Task completed successfully")
elif success is False:
    print("❌ Task failed or incomplete")
```

---

## 10. Common Mistakes When Accessing Outputs

### Mistake 1: Assuming fields always exist
```python
# WRONG - Will crash if models_usage is None
tokens = result.messages[0].models_usage.prompt_tokens

# RIGHT - Check first
msg = result.messages[0]
if msg.models_usage:
    tokens = msg.models_usage.prompt_tokens
else:
    tokens = 0
```

### Mistake 2: Not checking message type
```python
# WRONG - Will crash on non-TextMessage
for msg in result.messages:
    text = msg.content  # Not all messages have content!

# RIGHT - Check type or use getattr
for msg in result.messages:
    text = getattr(msg, 'content', '')
```

### Mistake 3: Assuming metadata exists
```python
# WRONG - Will crash if metadata is None
exec_time = result.metadata["execution_time"]

# RIGHT
if result.metadata:
    exec_time = result.metadata.get("execution_time")
```

### Mistake 4: Not handling empty messages
```python
# WRONG - Crashes if no messages
final = result.messages[-1]

# RIGHT
if result.messages:
    final = result.messages[-1]
else:
    print("No messages received")
```

---

## 11. Type Hints for Better IDE Support

```python
from autogen_agentchat.task import TaskResult
from autogen_agentchat.messages import TextMessage, HandoffMessage
from typing import Optional

def process_result(result: TaskResult) -> Optional[str]:
    """Extract final response from TaskResult"""
    
    if not result.messages:
        return None
    
    final_msg = result.messages[-1]
    
    if isinstance(final_msg, TextMessage):
        return final_msg.content
    
    return None

# Now your IDE provides autocomplete
result = await agent.run(task="...")
response = process_result(result)
```

---

## 12. Debugging Output Structures

### Pretty print any object
```python
import json
from dataclasses import asdict

def pretty_print_object(obj):
    """Print any AutoGen object nicely"""
    try:
        # Try Pydantic first
        if hasattr(obj, 'model_dump'):
            data = obj.model_dump()
        # Try dataclass
        elif hasattr(obj, '__dataclass_fields__'):
            data = asdict(obj)
        # Try dict conversion
        else:
            data = vars(obj)
        
        print(json.dumps(data, indent=2, default=str))
    except Exception as e:
        print(f"Cannot convert: {e}")
        print(repr(obj))

# Usage
pretty_print_object(result)
pretty_print_object(result.messages[0])
```

---

## 13. Version-Dependent Fields

**Important**: These field names and types may vary:

| Field | Version | Type | Notes |
|-------|---------|------|-------|
| `models_usage` | 0.2.x | ModelUsage \| None | May be unavailable |
| `stop_reason` | 0.2.x | str | Enum values vary |
| `metadata` | 0.2.x | dict \| None | Content provider-specific |
| `tool_calls` | 0.2.x | list[ToolCall] | Structure may change |

**Always verify**:
```python
# Check what's actually available
result = await agent.run(...)
print(dir(result))  # See all attributes
print(vars(result))  # See all values
```

---

## Summary: Understanding Any AutoGen Output

**Process**:
1. Get the object: `result = await agent.run(...)`
2. Check its type: `print(type(result))`
3. List attributes: `print(dir(result))`
4. Read values: `print(vars(result))`
5. Access safely: Check for None, use getattr
6. Inspect nested: Recurse for lists/dicts
7. Reference this doc: If unsure, look up the object

**Golden Rule**:
```python
# Always assume fields might not exist or be None
value = getattr(obj, 'field_name', default_value)
if obj.optional_field:  # If field is Optional
    use_it(obj.optional_field)
```

---

## Next Steps

Now that you understand runtime outputs:
- Read how to use TaskResult in practice
- Understand error handling (check stop_reason)
- Build robust output handling code

---

## References

- This document serves AutoGen 0.2.x
- All structures verified against current API
- For differences, run `dir()` and `vars()` on actual objects
- Check official AutoGen source for exact type definitions

---

## Version Notes

- **Verified for**: AutoGen 0.2.x
- **Field changes**: Between versions, fields may appear/disappear
- **If unsure**: Inspect object at runtime, don't assume
- **Type hints**: Use type hints for IDE support and safety
