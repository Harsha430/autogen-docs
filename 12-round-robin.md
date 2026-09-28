# Part 12: RoundRobinGroupChat - Fixed-Order Teams

## What It Is

**RoundRobinGroupChat** orchestrates agents in a fixed, repeating order.

```
Agent A → Agent B → Agent C → Agent A → ...
```

---

## When to Use

✅ Linear workflows where order is clear  
✅ Sequential processing (research → analyze → write)  
✅ Fixed stages that must happen in order  
✅ Simple multi-agent systems  

❌ When next agent depends on current conversation  
❌ When you need intelligent routing  
❌ When agents should choose their successors  

---

## Constructor

```python
from autogen_agentchat.teams import RoundRobinGroupChat

team = RoundRobinGroupChat(
    participants=[agent1, agent2, agent3],
    max_turns=10,
    termination_condition=None,
    model_client=None  # For team-level decisions
)
```

### Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| `participants` | list[Agent] | Agents in order |
| `max_turns` | int \| None | Max conversation turns |
| `termination_condition` | TerminationCondition \| None | Stop condition |
| `model_client` | ChatCompletionClient \| None | For team decisions |

---

## Execution Flow

```
Turn 1: Agent A speaks
  ↓ (A responds or calls tools)
Turn 2: Agent B speaks
  ↓ (B responds or calls tools)
Turn 3: Agent C speaks
  ↓ (C responds or calls tools)
Turn 4: Agent A speaks again (wraps around)
  ↓
... continues until max_turns or termination_condition
```

---

## Example: Research Pipeline

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.termination import MaxMessageTermination

# Define agents
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[web_search],
    system_message="Find information"
)

analyzer = AssistantAgent(
    name="analyzer",
    model_client=model_client,
    tools=[analyze_data],
    system_message="Synthesize findings"
)

writer = AssistantAgent(
    name="writer",
    model_client=model_client,
    tools=[format_report],
    system_message="Write report"
)

# Create team
team = RoundRobinGroupChat(
    participants=[researcher, analyzer, writer],
    max_turns=9,  # 3 agents × 3 rounds
    termination_condition=MaxMessageTermination(max_messages=50)
)

# Run
result = await team.run(task="Research Python 3.13")
```

### Execution Timeline

```
Turn 1: Researcher
  "I'll search for Python 3.13 features..."
  [web_search tool call]

Turn 2: Analyzer
  "Based on the research, here are key findings..."
  [analyze_data tool call]

Turn 3: Writer
  "Here's the report..."
  [format_report tool call]

Turn 4: Researcher
  "Is there anything else needed?"

Turn 5: Analyzer
  "The analysis is complete"

Turn 6: Writer
  "Report is ready"

→ Max turns reached → Stop
```

---

## Key Differences from Single Agent

### Single Agent
- One agent iterates with tools
- Can call same tool multiple times
- Unlimited tool iterations (up to max_tool_iterations)

### RoundRobin Team
- Multiple agents, each gets ONE turn
- Agents take turns speaking
- Limited by max_turns (not individual agent iterations)

---

## max_turns Explained

```python
team = RoundRobinGroupChat(
    participants=[A, B, C],
    max_turns=6  # Maximum total turns
)

Execution:
Turn 1: A speaks (1/6)
Turn 2: B speaks (2/6)
Turn 3: C speaks (3/6)
Turn 4: A speaks (4/6)
Turn 5: B speaks (5/6)
Turn 6: C speaks (6/6)
→ STOP (max_turns reached)
```

---

## Termination Conditions

### Stop on Message Content
```python
from autogen_agentchat.termination import StopMessageTermination

termination = StopMessageTermination(
    stop_words=["DONE", "COMPLETE", "FINISHED"]
)

# Execution stops if any agent says "DONE"
```

### Stop After N Messages
```python
termination = MaxMessageTermination(max_messages=50)

# Execution stops after 50 total messages
```

### Combined Conditions
```python
termination = MaxMessageTermination(max_messages=100)

team = RoundRobinGroupChat(
    participants=[...],
    max_turns=20,
    termination_condition=termination
)
# Stops if either max_turns OR max_messages is reached
```

---

## Common Use Cases

### Case 1: Document Processing Pipeline
```
Loader → Parser → Analyzer → Formatter
```

### Case 2: Research to Report
```
Researcher → Analyzer → Writer → Reviewer
```

### Case 3: Code Review
```
Coder → Tester → Reviewer
```

---

## Output Inspection

```python
result = await team.run(task="...")

print(f"Reason: {result.stop_reason}")
# "max_turns" or termination_condition result

print(f"Messages: {len(result.messages)}")
# Total messages from all agents

# See which agent spoke when
for i, msg in enumerate(result.messages):
    print(f"Turn {i+1}: {msg.source}")
```

---

## Common Mistakes

### Mistake 1: Too Few Turns
```python
# WRONG - Only allows 1 agent per round
team = RoundRobinGroupChat(
    participants=[A, B, C],
    max_turns=3  # A→B→C then stop
)

# RIGHT - Allow multiple rounds
team = RoundRobinGroupChat(
    participants=[A, B, C],
    max_turns=10  # 3+ complete rounds
)
```

### Mistake 2: Forgetting Termination
```python
# WRONG - Might run indefinitely
team = RoundRobinGroupChat(
    participants=[A, B, C],
    max_turns=None,  # No limit!
    termination_condition=None
)

# RIGHT - Always have a limit
team = RoundRobinGroupChat(
    participants=[A, B, C],
    max_turns=20,
    termination_condition=MaxMessageTermination(100)
)
```

---

## Next Steps

- [13-selector-group-chat.md](13-selector-group-chat.md) - Dynamic routing
- [14-swarm.md](14-swarm.md) - Handoff-based coordination

---

## References

- See [31-runtime-outputs-objects.md](31-runtime-outputs-objects.md) for TaskResult structure
- Official AutoGen: https://microsoft.github.io/autogen/
