# Part 5: Creating Custom Agents

## What It Is

**Custom agents** let you build specialized agents beyond `AssistantAgent` and `UserProxyAgent` by extending `BaseChatAgent`.

```python
class MyCustomAgent(BaseChatAgent):
    """Your specialized agent type"""
    
    async def on_messages(self, messages, sender):
        """Define how this agent processes messages"""
        pass
```

---

## When to Use

✅ Need special behavior not in standard agents  
✅ Custom message handling logic  
✅ Special tool integration  
✅ Domain-specific reasoning  

❌ Can achieve with system message (use AssistantAgent)  
❌ Just want different model (use model_client parameter)  

---

## BaseChatAgent: The Foundation

Every agent extends this:

```python
from autogen_agentchat.agents import BaseChatAgent
from autogen_agentchat.messages import TextMessage

class CustomAgent(BaseChatAgent):
    """Minimal custom agent"""
    
    def __init__(self, name: str):
        self._name = name
    
    @property
    def name(self) -> str:
        """Agent name (required)"""
        return self._name
    
    async def on_messages(
        self,
        messages: list,
        sender: str
    ) -> tuple[list, str]:
        """
        Process messages and return (response_messages, stop_reason)
        
        Args:
            messages: All messages so far
            sender: Name of agent that sent last message
        
        Returns:
            (response_messages, stop_reason)
        """
        # Your logic here
        response = TextMessage(
            content=f"I received message from {sender}",
            source=self.name
        )
        return [response], "message_processed"
```

### Required Properties

| Property | Type | Purpose |
|----------|------|---------|
| `name` | str | Unique agent identifier |

### Required Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `on_messages(messages, sender)` | (list, str) | Handle messages, return responses + stop_reason |

---

## Example 1: Logging Agent

```python
from autogen_agentchat.agents import BaseChatAgent
from autogen_agentchat.messages import TextMessage
import json
from datetime import datetime

class LoggingAgent(BaseChatAgent):
    """Agent that logs all messages to file"""
    
    def __init__(self, name: str, log_file: str = "agent.log"):
        self._name = name
        self.log_file = log_file
    
    @property
    def name(self) -> str:
        return self._name
    
    async def on_messages(self, messages: list, sender: str):
        """Log message and pass it through"""
        
        # Log to file
        latest_msg = messages[-1]
        log_entry = {
            "timestamp": datetime.now().isoformat(),
            "sender": sender,
            "content": latest_msg.content[:100]  # First 100 chars
        }
        
        with open(self.log_file, "a") as f:
            f.write(json.dumps(log_entry) + "\n")
        
        # Acknowledge
        response = TextMessage(
            content=f"Logged message from {sender}",
            source=self.name
        )
        return [response], "logged"

# Usage
logger = LoggingAgent("logger", log_file="conversation.log")
team = RoundRobinGroupChat(
    participants=[agent_a, logger, agent_b]
)
```

---

## Example 2: Validator Agent

```python
from autogen_agentchat.agents import BaseChatAgent
from autogen_agentchat.messages import TextMessage

class ValidatorAgent(BaseChatAgent):
    """Validates that responses meet criteria"""
    
    def __init__(self, name: str, min_length: int = 50):
        self._name = name
        self.min_length = min_length
    
    @property
    def name(self) -> str:
        return self._name
    
    async def on_messages(self, messages: list, sender: str):
        """Check if latest message is valid"""
        
        latest = messages[-1].content
        
        if len(latest) < self.min_length:
            response = TextMessage(
                content=f"Response too short ({len(latest)} < {self.min_length}). "
                       f"Please provide more detail.",
                source=self.name
            )
            return [response], "validation_failed"
        
        elif "TODO" in latest or "FIXME" in latest:
            response = TextMessage(
                content="Response contains incomplete items (TODO/FIXME). Fix before proceeding.",
                source=self.name
            )
            return [response], "incomplete"
        
        else:
            response = TextMessage(
                content="✓ Response validated successfully",
                source=self.name
            )
            return [response], "validated"

# Usage
validator = ValidatorAgent("validator", min_length=100)
team = RoundRobinGroupChat(
    participants=[writer, validator]
)
```

---

## Example 3: Filter Agent

```python
from autogen_agentchat.agents import BaseChatAgent
from autogen_agentchat.messages import TextMessage
import re

class FilterAgent(BaseChatAgent):
    """Filters out inappropriate content"""
    
    def __init__(self, name: str, blocked_words: list = None):
        self._name = name
        self.blocked_words = blocked_words or []
    
    @property
    def name(self) -> str:
        return self._name
    
    async def on_messages(self, messages: list, sender: str):
        """Check for blocked words"""
        
        latest = messages[-1].content.lower()
        
        for word in self.blocked_words:
            if word.lower() in latest:
                response = TextMessage(
                    content=f"Content contains blocked term: '{word}'. Cannot proceed.",
                    source=self.name
                )
                return [response], "blocked"
        
        response = TextMessage(
            content=f"✓ Content passed filtering",
            source=self.name
        )
        return [response], "approved"

# Usage
filter_agent = FilterAgent(
    "filter",
    blocked_words=["banned_term", "prohibited", "unsafe"]
)
```

---

## Handling Different Message Types

```python
from autogen_agentchat.messages import (
    TextMessage,
    HandoffMessage,
    ToolCallSummaryMessage
)

class SmartAgent(BaseChatAgent):
    """Handles different message types differently"""
    
    def __init__(self, name: str, model_client=None):
        self._name = name
        self.model_client = model_client
    
    @property
    def name(self) -> str:
        return self._name
    
    async def on_messages(self, messages: list, sender: str):
        """Handle based on message type"""
        
        latest = messages[-1]
        
        # Handle handoff request
        if isinstance(latest, HandoffMessage):
            response = TextMessage(
                content=f"Received handoff: {latest.content}",
                source=self.name
            )
            return [response], "acknowledged_handoff"
        
        # Handle tool results
        elif isinstance(latest, ToolCallSummaryMessage):
            response = TextMessage(
                content=f"Tool result: {latest.tool_call_result[:100]}",
                source=self.name
            )
            return [response], "processed_tool_result"
        
        # Handle text
        elif isinstance(latest, TextMessage):
            response = TextMessage(
                content=f"Processing: {latest.content[:50]}...",
                source=self.name
            )
            return [response], "processed"
        
        else:
            response = TextMessage(
                content=f"Unknown message type: {type(latest).__name__}",
                source=self.name
            )
            return [response], "unknown_type"
```

---

## State Management in Custom Agents

```python
from autogen_agentchat.agents import BaseChatAgent
from autogen_agentchat.messages import TextMessage

class StatefulAgent(BaseChatAgent):
    """Agent that tracks state across messages"""
    
    def __init__(self, name: str):
        self._name = name
        self.message_count = 0
        self.topics_seen = []
    
    @property
    def name(self) -> str:
        return self._name
    
    async def on_messages(self, messages: list, sender: str):
        """Track conversation state"""
        
        self.message_count += 1
        latest = messages[-1].content
        
        # Extract topic (first word)
        if latest:
            topic = latest.split()[0]
            if topic not in self.topics_seen:
                self.topics_seen.append(topic)
        
        response = TextMessage(
            content=f"Message #{self.message_count}. Topics seen: {self.topics_seen}",
            source=self.name
        )
        return [response], "state_updated"
    
    def get_state(self):
        """Get current state (for debugging)"""
        return {
            "message_count": self.message_count,
            "topics_seen": self.topics_seen
        }
```

---

## Common Mistakes

### Mistake 1: Forgetting to Return Tuple
```python
# WRONG
async def on_messages(self, messages, sender):
    response = TextMessage(content="Hello", source=self.name)
    return [response]  # ✗ Should return (messages, stop_reason)

# RIGHT
async def on_messages(self, messages, sender):
    response = TextMessage(content="Hello", source=self.name)
    return [response], "completed"  # ✓ Returns tuple
```

### Mistake 2: Not Setting Source
```python
# WRONG
response = TextMessage(content="Hello")  # ✗ No source

# RIGHT
response = TextMessage(content="Hello", source=self.name)
```

### Mistake 3: Forgetting @property
```python
# WRONG
def name(self):  # Not decorated
    return self._name

# RIGHT
@property
def name(self):  # ✓ Has @property
    return self._name
```

### Mistake 4: Blocking Operations
```python
# WRONG - Blocks the event loop
import time
async def on_messages(self, messages, sender):
    time.sleep(5)  # ✗ Blocks!
    return [response], "done"

# RIGHT - Use async
async def on_messages(self, messages, sender):
    await asyncio.sleep(5)  # ✓ Doesn't block
    return [response], "done"
```

---

## Complex Example: Decision Agent

```python
from autogen_agentchat.agents import BaseChatAgent
from autogen_agentchat.messages import TextMessage, HandoffMessage

class DecisionAgent(BaseChatAgent):
    """Routes based on message content"""
    
    def __init__(self, name: str):
        self._name = name
    
    @property
    def name(self) -> str:
        return self._name
    
    async def on_messages(self, messages: list, sender: str):
        """Decide what to do based on content"""
        
        latest = messages[-1].content.lower()
        
        # Route based on keywords
        if "budget" in latest or "cost" in latest or "price" in latest:
            handoff = HandoffMessage(
                to="finance_agent",
                content=f"User asking about costs: {messages[-1].content}",
                source=self.name
            )
            return [handoff], "routed_to_finance"
        
        elif "technical" in latest or "code" in latest or "error" in latest:
            handoff = HandoffMessage(
                to="tech_agent",
                content=f"User asking technical question: {messages[-1].content}",
                source=self.name
            )
            return [handoff], "routed_to_tech"
        
        else:
            response = TextMessage(
                content=f"I can help. Do you have questions about budget or technical issues?",
                source=self.name
            )
            return [response], "awaiting_clarification"
```

---

## Lifecycle Pattern

```
Team.run() called
  ↓
on_messages() called with full message history
  ↓
Your logic executes (can call tools, check conditions, etc.)
  ↓
Return (response_messages, stop_reason)
  ↓
Team processes response
  ↓
Repeat until termination_condition
```

---

## Testing Custom Agents

```python
import asyncio

async def test_custom_agent():
    """Test custom agent outside a team"""
    
    agent = MyCustomAgent("test")
    
    # Create test messages
    from autogen_agentchat.messages import TextMessage
    messages = [
        TextMessage(content="Hello", source="user")
    ]
    
    # Call on_messages directly
    responses, stop_reason = await agent.on_messages(messages, "user")
    
    print(f"Responses: {responses}")
    print(f"Stop reason: {stop_reason}")
    
    assert len(responses) > 0
    assert stop_reason in ["completed", "error", ...]

# Run test
asyncio.run(test_custom_agent())
```

---

## Next Steps

- [06-user-proxy-agent.md](06-user-proxy-agent.md) - Built-in human agent
- [07-messages.md](07-messages.md) - Message types in detail
- [11-teams.md](11-teams.md) - Using custom agents in teams

---

## References

- See [04-assistant-agent.md](04-assistant-agent.md) for standard agent
- Official AutoGen: https://microsoft.github.io/autogen/
