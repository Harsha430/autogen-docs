# Part 16: State Management

## What It Is

**State** is the internal condition of an agent or team at any moment. Understanding state helps you save/resume work and debug issues.

```
Turn 1: Agent has state_version=1, message_count=0
Turn 2: State changes to state_version=2, message_count=1
Turn 3: State changes again...
```

---

## Agent State vs Team State

### Agent State
- Internal variables (message count, context window)
- Tool call history
- Conversation history
- Custom attributes (your own tracking)

### Team State
- All participant agents' states
- Team-level termination condition
- Message history
- Execution phase

---

## State Lifecycle

```
Team created
  ↓ (initial state)
Task execution starts
  ↓ (state evolves)
Agent processes message
  ↓ (state changes)
Agent produces response
  ↓ (state updated)
Next turn
  ↓
Repeat until termination
  ↓
Final state captured
```

---

## What's Stored in State

### Message History
```python
# Every message ever generated
state.messages = [
    TextMessage("Hi", source="a"),
    TextMessage("Hello", source="b"),
    ...
]
```

### Agent Attributes
```python
# Per agent
state.agent_states = {
    "agent_a": {
        "name": "agent_a",
        "message_count": 5,
        "tools_called": ["search", "summarize"]
    },
    "agent_b": {...}
}
```

### Execution Context
```python
state.execution = {
    "current_turn": 3,
    "total_messages": 10,
    "started_at": "2024-01-01T10:00:00",
    "termination_condition": "not_met"
}
```

---

## State vs Memory vs History

| Concept | Scope | Purpose |
|---------|-------|---------|
| **State** | Current snapshot | Resume execution |
| **History** | Messages generated | Context for LLM |
| **Memory** | Long-term facts | Reference across runs |

```python
# STATE - Current condition
state = {
    "turn": 5,
    "messages_count": 20,
    "current_agent": "analyzer"
}

# HISTORY - What happened
history = [
    "Researcher found 3 sources",
    "Analyzer processed data",
    "Writer drafted report"
]

# MEMORY - Learned facts
memory = {
    "key_finding": "Revenue increased 15%",
    "risk": "Supply chain delay"
}
```

---

## State Consistency

### Consistency Challenge
```
If execution pauses mid-turn:
  Agent A has completed
  Agent B hasn't started yet
  
What's the "correct" state?
  Option 1: Before Agent B
  Option 2: After Agent B
```

### Checkpoint Approach
```python
# Save state at safe points (after agent completes)
async for event in team.run_stream(task="..."):
    if event["type"] == "message_created":
        # Safe point: agent completed
        save_state("checkpoint.json", event)
```

---

## Concurrent Execution & State

### Problem: Concurrent Calls
```python
# What if two teams run simultaneously?
task1 = team.run(task="A")
task2 = team.run(task="B")

results = await asyncio.gather(task1, task2)

# Each agent instance has ONE state
# Concurrent modifications conflict!
```

### Solution: Separate Instances
```python
# Create separate agents for each task
team1 = RoundRobinGroupChat(
    participants=[
        AssistantAgent("a1", model_client),
        AssistantAgent("b1", model_client)
    ]
)

team2 = RoundRobinGroupChat(
    participants=[
        AssistantAgent("a2", model_client),
        AssistantAgent("b2", model_client)
    ]
)

# Now concurrent run is safe
results = await asyncio.gather(
    team1.run(task="A"),
    team2.run(task="B")
)
```

---

## Debugging State

### Inspect State
```python
result = await team.run(...)

# Check messages
print(f"Total messages: {len(result.messages)}")

# Check which agents spoke
speakers = [m.source for m in result.messages]
from collections import Counter
print(Counter(speakers))

# Check stop reason
print(f"Stopped: {result.stop_reason}")
```

### State Inspection
```python
# In custom agent, track state
class StateDebugAgent(BaseChatAgent):
    def __init__(self, name):
        self._name = name
        self.turn_count = 0
    
    async def on_messages(self, messages, sender):
        self.turn_count += 1
        
        print(f"[State] Turn {self.turn_count}")
        print(f"  Message count: {len(messages)}")
        print(f"  Last sender: {sender}")
        
        response = TextMessage(
            content=f"Turn {self.turn_count}",
            source=self.name
        )
        return [response], "ok"
    
    @property
    def name(self):
        return self._name
```

### Export State for Analysis
```python
import json

def export_state(result, filepath):
    """Export execution state to file"""
    
    state_data = {
        "metadata": {
            "total_messages": len(result.messages),
            "stop_reason": result.stop_reason,
        },
        "messages": [
            {
                "source": m.source,
                "content": m.content[:100],
                "type": type(m).__name__
            }
            for m in result.messages
        ],
        "agent_participation": Counter([m.source for m in result.messages]).dict()
    }
    
    with open(filepath, "w") as f:
        json.dump(state_data, f, indent=2)

# Usage
export_state(result, "execution_state.json")
```

---

## State Persistence (Preview)

Basic state save/load (detailed in 19-state-persistence.md):

```python
# Save state
state_dict = {
    "messages": [m.model_dump() for m in result.messages],
    "metadata": {
        "stop_reason": result.stop_reason,
        "timestamp": datetime.now().isoformat()
    }
}

with open("checkpoint.json", "w") as f:
    json.dump(state_dict, f)

# Load state later
with open("checkpoint.json", "r") as f:
    saved_state = json.load(f)

# Continue from this state
# (specific mechanics in part 19)
```

---

## Stateless vs Stateful Design

### Stateless (Simple)
```python
# Each call is independent
for task in tasks:
    result = await agent.run(task)
    # No shared state between tasks
```

**Pros**: Simple, no state bugs  
**Cons**: Can't remember context across tasks

### Stateful (Complex)
```python
# Agent remembers context
state = {}
for task in tasks:
    result = await agent.run(task, previous_state=state)
    state = result.state  # Save for next task
```

**Pros**: Can build on previous tasks  
**Cons**: State bugs possible

---

## Common State Issues

### Issue 1: Unbounded Growth
```python
# WRONG - State grows forever
for i in range(1000):
    result = await agent.run(task=f"Task {i}")
    # Messages accumulate in memory
    total_messages = sum(len(r.messages) for r in results)  # Gets huge!

# RIGHT - Clear between runs
results = []
for i in range(1000):
    # Fresh agent for each task
    agent = AssistantAgent(f"agent_{i}", model_client)
    result = await agent.run(task=f"Task {i}")
    results.append(result)
```

### Issue 2: Shared State Mutation
```python
# WRONG - Multiple teams share agent
agent = AssistantAgent("shared", model_client)
team1 = RoundRobinGroupChat(participants=[agent, agent_b])
team2 = RoundRobinGroupChat(participants=[agent, agent_c])

# Both teams modify agent state!
asyncio.gather(team1.run(...), team2.run(...))  # Conflict

# RIGHT - Separate instances
team1 = RoundRobinGroupChat(
    participants=[AssistantAgent("a", mc), agent_b]
)
team2 = RoundRobinGroupChat(
    participants=[AssistantAgent("a", mc), agent_c]
)
```

---

## State Versioning (Preview)

When agent code changes, old state may be incompatible:

```python
# Version 1: Agent tracks messages
state_v1 = {
    "version": 1,
    "message_count": 10
}

# Version 2: Agent tracks messages + time
state_v2 = {
    "version": 2,
    "message_count": 10,
    "last_update": "2024-01-01"
}

# Need migration
def migrate_state(old_state):
    if old_state["version"] == 1:
        old_state["last_update"] = "unknown"
        old_state["version"] = 2
    return old_state
```

See part 19 for full state versioning patterns.

---

## Common Mistakes

### Mistake 1: Assuming State Persists
```python
# WRONG
result1 = await agent.run(task="A")
result2 = await agent.run(task="B")
# B doesn't remember A!

# RIGHT - If needed, restore state
agent.load_state(result1.state)
result2 = await agent.run(task="B")
```

### Mistake 2: Modifying State During Run
```python
# WRONG
async for event in team.run_stream(...):
    if event["type"] == "message_created":
        event["message"].content = "Hacked!"  # Modifying!

# RIGHT - Just read
async for event in team.run_stream(...):
    if event["type"] == "message_created":
        print(event["message"].content)  # Read only
```

### Mistake 3: State Not Thread-Safe
```python
# WRONG in multi-threaded code
shared_state = {}

def thread_1():
    shared_state["count"] = 1

def thread_2():
    shared_state["count"] = 2
    # Race condition!

# AutoGen is async, not threaded, so usually fine
# But be careful if mixing with threads
```

---

## Next Steps

- [17-model-context.md](17-model-context.md) - Context in state
- [19-state-persistence.md](19-state-persistence.md) - Save/load state
- [31-runtime-outputs-objects.md](31-runtime-outputs-objects.md) - TaskResult

---

## References

- Official AutoGen: https://microsoft.github.io/autogen/
