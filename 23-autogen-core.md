# Part 23: AutoGen Core - Low-Level Runtime

## What It Is

**AutoGen Core** is the underlying event-driven runtime that powers all teams and orchestration.

```
High-level (What you use):
  AssistantAgent, SelectorGroupChat, Swarm

Low-level (Under the hood):
  AgentRuntime, Topic-based pub/sub, Direct RPC
```

---

## AgentRuntime: The Foundation

### Architecture

```
AgentRuntime (orchestrator)
  ├─ Manages agent lifecycle
  ├─ Routes messages between agents
  ├─ Handles event streams
  └─ Manages subscriptions
```

### When to Use Core Directly

✅ Custom orchestration logic  
✅ Advanced routing needs  
✅ Fine-grained control over messaging  

❌ Standard teams sufficient  
❌ Simple workflows  

---

## Direct RPC vs Topic-Based

### Topic-Based (Standard)

```
Agent A broadcasts to "analysis" topic
  ↓
All subscribers to "analysis" receive message
  ↓
Agent B and C process
```

**Pros**: Loose coupling, scalable  
**Cons**: Less predictable routing

### Direct RPC (Explicit)

```
Agent A calls Agent B directly
  ↓
Only Agent B receives message
  ↓
Agent B responds directly
```

**Pros**: Predictable, simple  
**Cons**: Tight coupling

---

## Subscription Model

```python
# Agent subscribes to topics
runtime.subscribe(agent_b, topics=["analysis_results"])

# Agent A publishes
runtime.publish(
    message=result,
    topic="analysis_results",
    source=agent_a.name
)

# Agent B receives via subscription
```

---

## When to Use Core

### Example: Custom Orchestrator

```python
from autogen_agentchat.runtime import AgentRuntime

async def custom_orchestrator(agents, task):
    """Custom orchestration with Core"""
    
    runtime = AgentRuntime()
    
    # Custom routing logic
    for agent in agents:
        if agent.name == "decision_maker":
            # Decision maker listens to all
            runtime.subscribe(agent, topics=["*"])
        else:
            # Others only listen to their topic
            runtime.subscribe(agent, topics=[f"{agent.name}_result"])
    
    # Publish task
    await runtime.publish(
        message=task,
        topic="initial",
        source="system"
    )
    
    # Run until completion
    result = await runtime.wait_completion()
    return result
```

### Example: Advanced Routing

```python
async def advanced_routing(agents, task):
    """Route based on message content"""
    
    runtime = AgentRuntime()
    
    async def route_message(msg, source):
        """Custom router"""
        
        if "analysis" in msg.content:
            # Route to analyst
            await runtime.send_message_to(
                agent="analyst",
                message=msg,
                source=source
            )
        
        elif "approval" in msg.content:
            # Route to approver
            await runtime.send_message_to(
                agent="approver",
                message=msg,
                source=source
            )
        
        else:
            # Route to general handler
            await runtime.broadcast(message=msg)
    
    # Use custom router
    # ... (implementation)
```

---

## Event-Driven Architecture

```
Message published
  ↓ (event generated)
Event processed by runtime
  ↓ (subscriptions checked)
Listeners notified
  ↓ (agent on_messages called)
Agent produces response
  ↓ (new event)
```

---

## Low-Level Decorators

### @agent Decorator

```python
from autogen_agentchat.runtime import agent

# Old way (still works)
class MyAgent(BaseChatAgent):
    async def on_messages(self, messages, sender):
        pass

# New way (with decorator)
@agent
async def my_agent(messages, sender):
    # Simpler definition
    return [response], "ok"
```

---

## Direct Core Example

### Without High-Level Teams

```python
from autogen_agentchat.runtime import AgentRuntime

async def core_orchestration():
    """Use Core directly without teams"""
    
    runtime = AgentRuntime()
    
    # Create agents
    agent_a = MyAgent("a")
    agent_b = MyAgent("b")
    
    # Register with runtime
    await runtime.register_agent(agent_a)
    await runtime.register_agent(agent_b)
    
    # Subscribe to topics
    await runtime.subscribe(agent_b, topics=["from_a"])
    
    # Send message
    result = await runtime.send_message(
        target_agent=agent_a.name,
        message="Start process"
    )
    
    # Wait for completion
    final_result = await runtime.wait_completion()
    
    return final_result
```

---

## When Core is Better Than Teams

| Need | Use | Why |
|------|-----|-----|
| Custom routing | Core | Full control |
| Unique topology | Core | Can't model in teams |
| Broadcasting | Core | All receive message |
| Performance critical | Core | Minimal overhead |
| Standard workflow | Teams | Simpler API |
| Learning | Teams | Better documentation |

---

## Common Patterns

### Pattern 1: Fan-Out (One to Many)

```
Agent A publishes
  ↓
Agents B, C, D all receive (topic subscription)
```

### Pattern 2: Fan-In (Many to One)

```
Agents B, C, D publish to topic
  ↓
Agent A receives all (subscribed)
  ↓
Agent A aggregates
```

### Pattern 3: Pipeline

```
A → B → C → D
Sequential processing through Core
```

---

## Migration: Teams to Core

### Before (Teams)

```python
team = RoundRobinGroupChat(
    participants=[a, b, c]
)
result = await team.run(task="...")
```

### After (Core)

```python
runtime = AgentRuntime()
await runtime.register_agent(a)
await runtime.register_agent(b)
await runtime.register_agent(c)

# More control, but more verbose
result = await runtime.run(task="...")
```

---

## Next Steps

- [11-teams.md](11-teams.md) - High-level teams (easier)
- [25-complete-project.md](25-complete-project.md) - Production patterns
- Official AutoGen Core docs

---

## References

- Official AutoGen: https://microsoft.github.io/autogen/
