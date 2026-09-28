# Part 26: AutoGen vs LangGraph

## At a Glance

| Aspect | AutoGen | LangGraph |
|--------|---------|----------|
| **Design** | Agent-based orchestration | Graph/workflow-based |
| **State** | Per-message in team | Explicit state dict |
| **Routing** | Agent decisions or explicit | Graph nodes and edges |
| **Use case** | Agent teams & autonomy | Complex workflows |
| **Learning curve** | Medium | Steep |

---

## Architecture Differences

### AutoGen: Agent-Centric

```
Agent → Process → Decision → Agent → Process → Decision

Team orchestrates agents talking to each other
```

### LangGraph: Graph-Centric

```
Node1 → (condition) → Node2
           ↓
         Node3 → (condition) → Node4

Nodes are tasks, edges are transitions
```

---

## State Management

### AutoGen: Message-Based

```python
# State is implicit in message history
result = await team.run(task="...")
state = result.messages  # This is the state
```

### LangGraph: Explicit State

```python
# State is explicit dictionary
state = {
    "messages": [...],
    "count": 0,
    "metadata": {...}
}

# Node updates state explicitly
def my_node(state):
    state["count"] += 1
    return state
```

---

## Message Routing

### AutoGen Routing

**Option 1: Fixed Order**
```python
team = RoundRobinGroupChat(participants=[A, B, C])
# A → B → C → A → ...
```

**Option 2: Selector-Decided**
```python
team = SelectorGroupChat(
    participants=[A, B, C],
    selector_prompt="Who should speak?"
)
# Selector model decides
```

**Option 3: Agent-Decided**
```python
# Agent sends HandoffMessage(to="B")
# Swarm routes based on agent choice
```

### LangGraph Routing

```python
# Explicit graph with conditions
graph.add_node("research", research_node)
graph.add_node("analyze", analyze_node)
graph.add_node("review", review_node)

graph.add_edge("research", "analyze")
graph.add_conditional_edges(
    "analyze",
    lambda x: "review" if x["score"] > 0.8 else "research"
)

# Clear, deterministic flow
```

---

## When to Use Each

### Use AutoGen If:

✅ Agent autonomy important  
✅ Multiple decision-makers in workflow  
✅ Natural agent-to-agent communication  
✅ Building multi-agent systems  
✅ Focus on "who talks next"  

**Example**: 
- Research team where any agent can handoff to specialist
- Collaborative systems (devs, testers, reviewers)

### Use LangGraph If:

✅ Complex workflow with many steps  
✅ Clear, deterministic flow  
✅ Need explicit state management  
✅ Debugging/visibility critical  
✅ Focus on "what step next"  

**Example**:
- ETL pipeline: Extract → Clean → Transform → Load
- Approval workflow: Submit → Review → Approve → Execute

---

## Example: Same Task, Different Approaches

### Task: Code Review Workflow

#### AutoGen Approach

```python
from autogen_agentchat.teams import SelectorGroupChat

developer = AssistantAgent(name="developer", ...)
tester = AssistantAgent(name="tester", ...)
approver = AssistantAgent(name="approver", ...)

team = SelectorGroupChat(
    participants=[developer, tester, approver],
    selector_prompt="Who should review next?"
)

result = await team.run(task="Review this code")
```

**Flow**: Selector decides who reviews → dynamic routing

#### LangGraph Approach

```python
from langgraph.graph import StateGraph, START, END

def developer_node(state):
    # Code changes
    state["code"] = "..."
    return state

def test_node(state):
    # Run tests
    state["tests_passed"] = True
    return state

def approve_node(state):
    # Approve
    state["approved"] = True
    return state

graph = StateGraph(dict)
graph.add_node("developer", developer_node)
graph.add_node("test", test_node)
graph.add_node("approve", approve_node)

graph.add_edge(START, "developer")
graph.add_edge("developer", "test")
graph.add_conditional_edges(
    "test",
    lambda x: "approve" if x["tests_passed"] else "developer"
)
graph.add_edge("approve", END)
```

**Flow**: Clear stages → defined transitions

---

## Message vs State

### AutoGen: Focus on Messages

```python
# Messages are the state
for msg in result.messages:
    print(f"{msg.source}: {msg.content}")
    
# Implicit context from message history
```

### LangGraph: Focus on State

```python
# State is explicit
state = {
    "messages": [...],
    "analysis_result": "...",
    "approved": False,
    "error_count": 0
}

# Every node can access and modify state
```

---

## Debugging & Visibility

### AutoGen: Message Timeline

```
Turn 1: agent_a spoke
Turn 2: agent_b spoke
Turn 3: agent_a spoke
...
```

### LangGraph: Node Execution

```
Entered: node_research
  State: {messages: [...]}
  Updated: analysis_result = "..."
  
Entered: node_review
  State: {messages: [...], analysis_result: "..."}
  Updated: approved = true
  
Exited graph
```

---

## Tool Integration

### AutoGen

```python
agent = AssistantAgent(
    name="agent",
    model_client=client,
    tools=[tool1, tool2]
)
# Agent decides when to call tools
```

### LangGraph

```python
def tool_node(state):
    """Explicit tool calling node"""
    result = tool1(state["input"])
    state["tool_result"] = result
    return state

# Tool calling is explicit node in graph
```

---

## Scalability

| Aspect | AutoGen | LangGraph |
|--------|---------|----------|
| **Many agents** | ✅ Good | ⚠️ Many nodes |
| **Complex flow** | ⚠️ Difficult | ✅ Clear |
| **Debugging** | Medium | Good |
| **Performance** | Good | Excellent |

---

## Migration Guide: AutoGen → LangGraph

### Step 1: Map Agents to Nodes

```python
# AutoGen
team = RoundRobinGroupChat(
    participants=[researcher, analyzer, writer]
)

# LangGraph
graph.add_node("research", research_node)
graph.add_node("analyze", analyze_node)
graph.add_node("write", write_node)
```

### Step 2: Map Routing to Edges

```python
# AutoGen (Selector decides)
# LangGraph (explicit edges)
graph.add_edge("research", "analyze")
graph.add_edge("analyze", "write")
```

### Step 3: Extract State Management

```python
# AutoGen (implicit)
state = result.messages

# LangGraph (explicit)
state = {
    "messages": [...],
    "research_data": "...",
    "analysis": "..."
}
```

---

## Combining Both Frameworks

### Hybrid Approach

```python
# Use LangGraph for high-level flow
# Use AutoGen for agent team within a node

def agent_team_node(state):
    """Run AutoGen team within LangGraph node"""
    
    team = RoundRobinGroupChat(participants=[a, b, c])
    result = await team.run(task=state["task"])
    
    state["agent_result"] = result.messages[-1].content
    return state

# Now integrate in LangGraph
graph.add_node("agent_phase", agent_team_node)
```

---

## Common Mistakes When Switching

### Mistake 1: Forgetting State is Explicit
```python
# WRONG - Implicit assumption
def my_node(state):
    return state  # Nothing changed!

# RIGHT - Explicit update
def my_node(state):
    state["result"] = compute_result()
    return state
```

### Mistake 2: Loose State Structure
```python
# WRONG - No schema
state = {}  # Anything can go in!

# RIGHT - Define schema
class MyState(BaseModel):
    messages: list
    analysis: str
    approved: bool
```

### Mistake 3: Missing Conditional Edges
```python
# WRONG - Only straight edges
graph.add_edge("node_a", "node_b")  # Always goes to B

# RIGHT - Conditional routing
graph.add_conditional_edges(
    "node_a",
    lambda x: "node_b" if x["condition"] else "node_c"
)
```

---

## Summary: Choose Based on Your Need

| If You Need | Choose | Why |
|------------|--------|-----|
| Agent autonomy | AutoGen | Agents make routing decisions |
| Clear workflow | LangGraph | Graph makes flow obvious |
| Multi-agent team | AutoGen | Team orchestration built-in |
| Complex conditions | LangGraph | Explicit conditional edges |
| Quick prototype | AutoGen | Less boilerplate |
| Production workflow | LangGraph | Better debuggability |

---

## Next Steps

- [27-autogen-vs-crewai.md](27-autogen-vs-crewai.md) - Compare with CrewAI
- [25-complete-project.md](25-complete-project.md) - Full AutoGen project
- LangGraph docs: https://langchain-ai.github.io/langgraph

---

## References

- LangGraph: https://langchain-ai.github.io/langgraph
- AutoGen: https://microsoft.github.io/autogen/
