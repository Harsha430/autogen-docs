# Part 7: Messages - Communication in AutoGen

## What It Is

**Messages** are the currency of agent communication. Every agent response is one or more messages.

```
Agent A speaks → TextMessage
                     ↓
            Team passes to Agent B
                     ↓
Agent B reads TextMessage, responds → TextMessage or HandoffMessage
```

---

## Message Hierarchy

```
Message (base class)
├── TextMessage         - Text responses
├── HandoffMessage      - Transfer to another agent
├── ToolCallSummaryMessage  - Tool execution result
└── CustomMessage       - Your own message type
```

---

## TextMessage: Standard Communication

```python
from autogen_agentchat.messages import TextMessage

# Create
msg = TextMessage(
    content="I've analyzed the request",
    source="analyst"
)

# Properties
print(msg.content)  # "I've analyzed the request"
print(msg.source)   # "analyst"
```

### Structure

```python
TextMessage(
    content: str,         # Message text
    source: str = None,   # Sender name
    models_used: list = None  # Optional tracking
)
```

### Usage

```python
# In agent's on_messages()
response = TextMessage(
    content="I've completed the analysis",
    source=self.name
)
return [response], "completed"

# In team execution
for msg in result.messages:
    if isinstance(msg, TextMessage):
        print(f"{msg.source}: {msg.content}")
```

---

## HandoffMessage: Transferring Control

```python
from autogen_agentchat.messages import HandoffMessage

# Create
handoff = HandoffMessage(
    to="specialist",
    content="This needs specialized handling",
    source="router"
)

# Properties
print(handoff.to)       # "specialist" (target agent)
print(handoff.content)  # Explanation
print(handoff.source)   # "router" (who initiated)
```

### Structure

```python
HandoffMessage(
    to: str,              # Target agent name
    content: str,         # Why/what to handoff
    source: str = None,   # Sending agent
)
```

### Usage in Agents

```python
# In custom agent or Swarm
if specialized:
    handoff = HandoffMessage(
        to="specialist_agent",
        content=f"Task details: {task_description}",
        source=self.name
    )
    return [handoff], "handed_off"
```

---

## ToolCallSummaryMessage: Tool Results

```python
from autogen_agentchat.messages import ToolCallSummaryMessage

# Created by agent after tool execution
msg = ToolCallSummaryMessage(
    tool_name="search_web",
    tool_call_id="call_123",
    tool_call_args={"query": "Python asyncio"},
    tool_call_result="Found 500 results..."
)

# Properties
print(msg.tool_name)          # "search_web"
print(msg.tool_call_args)     # {"query": "..."}
print(msg.tool_call_result)   # Result text
```

### Structure

```python
ToolCallSummaryMessage(
    tool_name: str,           # Name of tool
    tool_call_id: str,        # Unique call ID
    tool_call_args: dict,     # Arguments passed
    tool_call_result: str,    # Result/output
    source: str = None,       # Agent that called
)
```

### Usage in Message Handling

```python
# In on_messages(), detect tool results
for msg in messages:
    if isinstance(msg, ToolCallSummaryMessage):
        print(f"Tool {msg.tool_name} result: {msg.tool_call_result}")
```

---

## Message Lifecycle

```
Agent starts
  ↓
Receives all previous messages
  ↓
on_messages(messages, sender) called
  ↓
Agent creates response message(s)
  ↓
Returns (messages_list, stop_reason)
  ↓
Team adds to conversation history
  ↓
Next agent receives all messages so far
```

### Example Flow

```
Turn 1: Agent A
  messages = []  (empty, first turn)
  → Creates TextMessage("Hello")
  
Turn 2: Agent B
  messages = [TextMessage("Hello", source="a")]
  → Reads Agent A's message
  → Creates TextMessage("Hi back")
  
Turn 3: Agent A
  messages = [
    TextMessage("Hello", source="a"),
    TextMessage("Hi back", source="b")
  ]
  → Reads full conversation
  → Creates TextMessage("Great!")
```

---

## Type Checking and Filtering

### Check Message Type

```python
from autogen_agentchat.messages import (
    TextMessage,
    HandoffMessage,
    ToolCallSummaryMessage
)

def process_message(msg):
    if isinstance(msg, TextMessage):
        return f"Text: {msg.content}"
    
    elif isinstance(msg, HandoffMessage):
        return f"Handoff to {msg.to}"
    
    elif isinstance(msg, ToolCallSummaryMessage):
        return f"Tool result: {msg.tool_call_result}"
    
    else:
        return "Unknown"
```

### Filter Messages

```python
# Get only text messages
text_msgs = [m for m in messages if isinstance(m, TextMessage)]

# Get tool results
tool_results = [m for m in messages if isinstance(m, ToolCallSummaryMessage)]

# Get last message from specific agent
last_from_agent_a = next(
    (m for m in reversed(messages) if m.source == "agent_a"),
    None
)
```

---

## Custom Message Types

### Define Custom Message

```python
from autogen_agentchat.messages import Message
from dataclasses import dataclass

@dataclass
class StatusMessage(Message):
    """Custom status update"""
    status: str
    progress: int  # 0-100
    source: str
    
    def __str__(self):
        return f"[{self.status}] {self.progress}%"
```

### Use Custom Message

```python
class ProgressAgent(BaseChatAgent):
    """Tracks progress with custom messages"""
    
    async def on_messages(self, messages, sender):
        # Create custom message
        status = StatusMessage(
            status="processing",
            progress=75,
            source=self.name
        )
        return [status], "progress_update"

# In team, handle custom messages
for msg in result.messages:
    if isinstance(msg, StatusMessage):
        print(f"{msg.status}: {msg.progress}%")
```

---

## Message Content Best Practices

### Write Clear Content

```python
# WRONG - Vague
TextMessage(content="OK", source="agent")

# RIGHT - Clear and specific
TextMessage(
    content="Analysis complete. Found 3 key insights: 1) Cost increase 15%, 2) Timeline delayed 2 weeks, 3) Resource shortage in Q2",
    source="analyst"
)
```

### Include Context

```python
# WRONG - Isolated response
TextMessage(content="Yes", source="reviewer")

# RIGHT - Includes reason
TextMessage(
    content="APPROVED - Code follows all style guidelines and passes test coverage threshold",
    source="code_reviewer"
)
```

### Structure for Parsing

```python
# WRONG - Ambiguous
TextMessage(content="Need to revise more detail needed", source="agent")

# RIGHT - Machine-parseable
TextMessage(
    content="REVISION NEEDED: The analysis lacks detail. Specifically: 1) Add cost breakdown, 2) Include timeline, 3) Add risk assessment",
    source="agent"
)
```

---

## Message Inspection Patterns

### Count Message Types

```python
from collections import Counter

def analyze_messages(messages):
    types = [type(m).__name__ for m in messages]
    counter = Counter(types)
    print(counter)
    # Counter({'TextMessage': 15, 'ToolCallSummaryMessage': 3, 'HandoffMessage': 1})
```

### Track Conversation Flow

```python
def show_conversation(messages):
    for i, msg in enumerate(messages):
        if isinstance(msg, TextMessage):
            print(f"{i}: {msg.source} → {msg.content[:50]}...")
        
        elif isinstance(msg, HandoffMessage):
            print(f"{i}: {msg.source} HANDS OFF TO {msg.to}")
        
        elif isinstance(msg, ToolCallSummaryMessage):
            print(f"{i}: {msg.source} called {msg.tool_name}()")
```

### Extract Key Information

```python
def get_final_decision(messages):
    """Get last text message (final decision)"""
    for msg in reversed(messages):
        if isinstance(msg, TextMessage):
            return msg.content
    return None

def get_tool_calls(messages):
    """List all tools called"""
    tools = []
    for msg in messages:
        if isinstance(msg, ToolCallSummaryMessage):
            tools.append((msg.tool_name, msg.tool_call_args))
    return tools

def get_handoff_chain(messages):
    """Trace handoffs"""
    chain = []
    for msg in messages:
        if isinstance(msg, HandoffMessage):
            chain.append((msg.source, msg.to))
    return chain
```

---

## Common Mistakes

### Mistake 1: Missing source
```python
# WRONG
msg = TextMessage(content="Hello")  # Who sent this?

# RIGHT
msg = TextMessage(content="Hello", source=self.name)
```

### Mistake 2: Wrong Message Type
```python
# WRONG - Using TextMessage for handoff
msg = TextMessage(
    content="I'm handing off to specialist",
    source=self.name
)

# RIGHT - Use HandoffMessage
msg = HandoffMessage(
    to="specialist",
    content="Please take it from here",
    source=self.name
)
```

### Mistake 3: Incorrect Handoff Target
```python
# WRONG - Agent name has typo
msg = HandoffMessage(
    to="specailist",  # Typo!
    content="Help",
    source=self.name
)

# RIGHT
msg = HandoffMessage(
    to="specialist",  # Correct name
    content="Help",
    source=self.name
)
```

### Mistake 4: Not Handling Tool Failures
```python
# WRONG - Ignores errors
for msg in messages:
    if isinstance(msg, ToolCallSummaryMessage):
        print(msg.tool_call_result)  # Might be error!

# RIGHT - Check for errors
for msg in messages:
    if isinstance(msg, ToolCallSummaryMessage):
        if "error" in str(msg.tool_call_result).lower():
            print(f"Tool failed: {msg.tool_call_result}")
        else:
            print(f"Tool succeeded: {msg.tool_call_result}")
```

---

## Message Storage and Serialization

### Save Messages to File

```python
import json

result = await team.run(...)

# Serialize
messages_data = [msg.model_dump() for msg in result.messages]

with open("conversation.json", "w") as f:
    json.dump(messages_data, f, indent=2)

# Load
with open("conversation.json", "r") as f:
    loaded_data = json.load(f)
    # Note: Need to reconstruct message objects from dict
```

---

## Next Steps

- [08-run-and-streaming.md](08-run-and-streaming.md) - Execution and events
- [10-single-agent-loop.md](10-single-agent-loop.md) - Agent execution
- [31-runtime-outputs-objects.md](31-runtime-outputs-objects.md) - TaskResult

---

## References

- See [04-assistant-agent.md](04-assistant-agent.md) for agent basics
- See [05-custom-agents.md](05-custom-agents.md) for on_messages() details
- Official AutoGen: https://microsoft.github.io/autogen/
