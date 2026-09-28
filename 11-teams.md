# Part 11: Teams - Multi-Agent Orchestration Base

## What is a Team?

A **team** coordinates multiple agents working together. Each agent processes tasks, communicates with others, and the team manages the orchestration logic.

```
Team = Container for multiple agents
       + Orchestration rules
       + Message routing
       + Termination logic
```

---

## Why Teams Exist

### Problem: Single Agent Limitations
```
Task: "Research and write a report"

Single agent:
- Researches
- Analyzes
- Writes
- Reviews

= One agent doing everything (suboptimal)
```

### Solution: Specialized Teams
```
Task: "Research and write a report"

Multi-agent team:
Researcher agent → Analyzer agent → Writer agent → Reviewer agent
     ↓
Each agent specializes
Communication is explicit
Clear handoffs
```

---

## BaseGroupChat: The Foundation

**All team types inherit from BaseGroupChat.**

```python
from autogen_agentchat.teams import BaseGroupChat

class CustomTeam(BaseGroupChat):
    """Your custom orchestration logic"""
    pass
```

### BaseGroupChat Interface

```python
class BaseGroupChat:
    """Base class for all team types"""
    
    def __init__(
        self,
        participants: list[Agent],
        termination_condition: TerminationCondition | None = None,
        max_turns: int | None = None,
        # ... other params
    ):
        pass
    
    async def run(
        self,
        task: str,
        cancellation_token: CancellationToken | None = None
    ) -> TaskResult:
        """Run team to completion"""
        pass
    
    async def run_stream(
        self,
        task: str
    ) -> AsyncIterator:
        """Stream events during execution"""
        pass
    
    async def reset(self) -> None:
        """Reset team state"""
        pass
```

---

## Key Concepts

### 1. Participants (Agents)
```python
team = RoundRobinGroupChat(
    participants=[
        researcher_agent,    # Agent 1
        analyzer_agent,      # Agent 2
        writer_agent         # Agent 3
    ]
)
```

All agents participate in execution.

### 2. Termination Conditions
```python
from autogen_agentchat.termination import MaxMessageTermination

termination = MaxMessageTermination(max_messages=100)

team = RoundRobinGroupChat(
    participants=[...],
    termination_condition=termination
)
```

Stops execution when condition is met.

### 3. Message Routing
Different team types route messages differently:
- **RoundRobin**: Fixed order (A → B → C → A)
- **Selector**: Dynamic selection (Selector picks next agent)
- **Swarm**: Handoff-based (Agents transfer control)

---

## Team Output

**Teams return TaskResult** (same as single agents):

```python
result = await team.run(task="...")

# Same structure
result.messages      # All messages from all agents
result.stop_reason   # Why it stopped
result.metadata      # Execution metadata

# But messages from MULTIPLE agents
for msg in result.messages:
    print(f"{msg.source}: {msg.content[:50]}")
    # Output:
    # researcher: Found information about...
    # analyzer: Synthesis of findings...
    # writer: Here is the report...
```

---

## Team Types Overview

| Type | Routing | When to Use |
|------|---------|------------|
| **RoundRobin** | Fixed order (A→B→C→A) | Linear workflows |
| **Selector** | Dynamic selection | Complex reasoning |
| **Swarm** | Handoff-based | Autonomous collaboration |

---

## Common Team Patterns

### Pattern 1: Sequential Processing
```
Input → Agent1 → Agent2 → Agent3 → Output
```

**Use RoundRobinGroupChat** with max_turns.

### Pattern 2: Hierarchical Review
```
Input → Processor → Reviewer → ✓/✗ → (if ✗ loop back)
```

**Use SelectorGroupChat** with reviewer decision logic.

### Pattern 3: Autonomous Collaboration
```
Agent A
  ↓ handoff
Agent B
  ↓ handoff
Agent C (completes)
```

**Use Swarm** with handoff rules.

---

## Next Steps

Choose your team type:
- **[12-round-robin.md](12-round-robin.md)** - Fixed order
- **[13-selector-group-chat.md](13-selector-group-chat.md)** - Dynamic selection
- **[14-swarm.md](14-swarm.md)** - Handoff-based

---

## Version Notes

- **Current**: AutoGen 0.2.x
- BaseGroupChat is stable base
- Team implementations may vary
