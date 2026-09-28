# Part 13: SelectorGroupChat - Dynamic Speaker Selection

## What It Is

**SelectorGroupChat** uses an AI agent to decide which team member speaks next based on conversation context.

```
Current Conversation → Selector Agent → "Agent B should speak next"
                                              ↓
                                        Agent B speaks
```

Unlike RoundRobin (fixed order), Selector adapts the speaker based on what's being discussed.

---

## When to Use

✅ When next speaker logically depends on the conversation  
✅ Intelligent routing through specialized agents  
✅ Smart review processes (find right reviewer)  
✅ Complex workflows needing adaptive routing  

❌ For simple sequential tasks (use RoundRobin)  
❌ When order is predetermined  
❌ For basic handoff workflows (use Swarm)  

---

## Constructor

```python
from autogen_agentchat.teams import SelectorGroupChat

team = SelectorGroupChat(
    participants=[agent1, agent2, agent3],
    model_client=model_client,
    selector_prompt="Choose the next speaker...",
    max_turns=10,
    termination_condition=None
)
```

### Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| `participants` | list[Agent] | Available team members |
| `model_client` | ChatCompletionClient | **REQUIRED** - Makes routing decisions |
| `selector_prompt` | str \| None | How to select next speaker |
| `max_turns` | int \| None | Max conversation turns |
| `termination_condition` | TerminationCondition \| None | Stop condition |

**Critical**: `model_client` is required for Selector (not optional like RoundRobin).

---

## How It Works

### Step 1: Conversation Happens
Agent speaks, may call tools, produces output.

### Step 2: Selector Makes Decision
Model client evaluates: "Given the current conversation, who should speak next?"

### Step 3: Next Agent Speaks
Selector's choice becomes the new speaker.

### Step 4: Repeat
Until max_turns or termination condition.

```
Turn 1: Agent A speaks
        [Team passes conversation to selector model]
Turn 2: Selector chooses → Agent C
        Agent C speaks
Turn 3: Selector chooses → Agent A
        Agent A speaks
...
```

---

## Selector Prompt Design

The `selector_prompt` is critical. It tells the model:
- Who the participants are
- What each specializes in
- How to choose among them

### Example: Smart Code Review

```python
selector_prompt = """
You are a code review orchestrator. Choose the next speaker from:
- coder: Implements features and fixes bugs
- tester: Verifies correctness and edge cases
- designer: Ensures architecture and design patterns
- reviewer: Makes final approval/rejection decisions

Current task: Review a pull request for a payment system.

Given the conversation so far, who should speak next?
Respond with ONLY the agent name.
"""

team = SelectorGroupChat(
    participants=[coder, tester, designer, reviewer],
    model_client=model_client,
    selector_prompt=selector_prompt,
    max_turns=12
)
```

### Prompt Design Tips

✅ List all participants clearly  
✅ Describe each participant's role  
✅ Give examples of when each should speak  
✅ End with clear instruction ("Respond with ONLY the name")  
✅ Include context: What is the current task?  

❌ Ambiguous descriptions  
❌ Forgetting to list all participants  
❌ No clear output format  
❌ Conflicting role definitions  

---

## Roles and Participants Concept

Participants should have clear, non-overlapping responsibilities:

### Bad: Overlapping Roles
```python
participants=[
    analyst,      # "Analyzes data"
    analyzer,     # "Analyzes patterns"  ← Confusing!
    interpreter   # "Interprets results"
]
```

The selector can't distinguish which to choose.

### Good: Clear Specialization
```python
participants=[
    data_engineer,    # "Retrieves and cleans data"
    statistician,     # "Performs statistical analysis"
    business_analyst, # "Interprets for business impact"
    communicator      # "Writes executive summary"
]
```

Selector can clearly route: data → statistics → business → communication.

---

## Example: Content Review Pipeline

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import SelectorGroupChat

# Create specialized agents
writer = AssistantAgent(
    name="writer",
    model_client=model_client,
    system_message="You are a writer. Draft engaging content."
)

editor = AssistantAgent(
    name="editor",
    model_client=model_client,
    system_message="You are an editor. Review for clarity and grammar."
)

fact_checker = AssistantAgent(
    name="fact_checker",
    model_client=model_client,
    system_message="You are a fact-checker. Verify accuracy."
)

approver = AssistantAgent(
    name="approver",
    model_client=model_client,
    system_message="You are an approver. Make final decisions."
)

# Create selector
selector_prompt = """
You manage a content review pipeline. Choose the next speaker:

- writer: Drafts initial content
- editor: Reviews for grammar, clarity, style
- fact_checker: Verifies all claims are accurate
- approver: Makes final publication decision

Current status: {current_task}

Who should speak next?
Respond with ONLY the agent name (writer/editor/fact_checker/approver).
"""

team = SelectorGroupChat(
    participants=[writer, editor, fact_checker, approver],
    model_client=model_client,
    selector_prompt=selector_prompt,
    max_turns=15
)

result = await team.run(
    task="Review and approve blog post about AI safety"
)
```

### Execution Flow

```
Turn 1: Writer
  "I'll draft a blog post..."
  [Produces: "AI Safety: Why We Need Guardrails..."]

Turn 2: Selector chooses → Editor
  "Grammar looks good. I'd reorganize the sections..."
  [Produces: revised structure]

Turn 3: Selector chooses → Fact-checker
  "Checking claims... Found issue: Citation X is outdated"
  [Produces: corrected citations]

Turn 4: Selector chooses → Writer (loops back)
  "Updating with correct citations..."

Turn 5: Selector chooses → Approver
  "All checks passed. Approved for publication!"
  
→ Task complete
```

---

## Prevention of Infinite Loops

SelectorGroupChat can create infinite loops if not carefully designed.

### Problem: Ping-Pong Loop
```
Turn 1: A speaks
Turn 2: B speaks
Turn 3: A speaks  ← Selector keeps choosing A and B
Turn 4: B speaks  ← Never progresses
```

### Solutions

**1. Progress-Based Routing**
```python
selector_prompt = """
...
Each agent should move the task forward. If we're repeating the same agents,
the current agent should produce a final output or move to the next stage.

Who should speak next? Don't repeat the same agent unless necessary.
"""
```

**2. Stage Markers**
```python
# In agent system messages
system_message = """
You are the editor. If you've finished edits:
1. Summarize changes
2. Say "Content ready for approval"
3. The selector will route to approver
"""
```

**3. Strict max_turns**
```python
team = SelectorGroupChat(
    participants=[...],
    max_turns=10,  # Hard limit prevents infinite loops
    termination_condition=MaxMessageTermination(max_messages=50)
)
```

**4. Watch for Patterns**
```python
result = await team.run(...)

# Inspect messages
speakers = [msg.source for msg in result.messages]
print(speakers)  # ["writer", "editor", "writer", "editor", "writer"]
# ↑ Pattern suggests infinite loop - redesign selector_prompt
```

---

## Common Mistakes

### Mistake 1: Selector Prompt Too Vague
```python
# WRONG
selector_prompt = "Choose the next person."

# RIGHT
selector_prompt = """
You manage a code review. Choose next:
- developer: Writes and fixes code
- tester: Verifies functionality
- architect: Reviews design

Developer just submitted code. Who reviews first?
Respond with ONLY: developer/tester/architect
"""
```

### Mistake 2: Not Setting model_client
```python
# WRONG - Missing model_client
team = SelectorGroupChat(
    participants=[A, B, C]
    # selector_prompt="..."
)

# RIGHT
team = SelectorGroupChat(
    participants=[A, B, C],
    model_client=model_client,  # Required!
    selector_prompt="..."
)
```

### Mistake 3: Overlapping Participant Roles
```python
# WRONG - Ambiguous
participants=[analyzer, data_analyst, stats_expert]

# RIGHT - Clear distinction
participants=[
    data_engineer,  # Gets raw data
    analyst,        # Analyzes patterns
    communicator    # Explains to stakeholders
]
```

### Mistake 4: No Progress Tracking
```python
# WRONG - Can loop forever
result = await team.run(task="Optimize code")

# RIGHT - Monitor for loops
speakers = [msg.source for msg in result.messages]
if len(set(speakers)) < 2:  # Only 1 unique agent
    print("Warning: Possible infinite loop")
```

---

## Limitations and When to Avoid

### When Selector is OVERKILL
- **Simple workflows**: Use RoundRobin instead
  ```python
  # Too complex:
  team = SelectorGroupChat(participants=[data_loader, processor])
  
  # Better:
  team = RoundRobinGroupChat(participants=[data_loader, processor])
  ```

- **Real-time constraints**: Selector adds model call overhead
  ```python
  # Selector calls model for EACH turn's routing decision
  # This is expensive for 50-turn conversations
  ```

- **Deterministic workflows**: Use Swarm with explicit handoffs
  ```python
  # If routing is predetermined, Swarm is clearer:
  agent_a.handoffs = [agent_b]
  agent_b.handoffs = [agent_c]
  ```

### When Selector SHINES
- Complex decision logic ("Which reviewer for payment code?")
- Adaptive workflows that learn from conversation
- Multi-stage processes with intelligent routing
- Human-like "let's ask the expert" decisions

---

## Output Inspection

```python
result = await team.run(task="...")

print(f"Stop reason: {result.stop_reason}")
# "max_turns" or other condition

print(f"Total messages: {len(result.messages)}")

# See routing decisions
for i, msg in enumerate(result.messages):
    print(f"Turn {i+1}: {msg.source} spoke")

# Find routing patterns
speakers = [msg.source for msg in result.messages]
print(f"Speaker sequence: {' → '.join(speakers)}")
```

---

## Comparison: RoundRobin vs Selector vs Swarm

| Feature | RoundRobin | Selector | Swarm |
|---------|-----------|----------|-------|
| **Order** | Fixed cycle | AI-decided | Agent-decided |
| **Routing Logic** | Built-in | Custom prompt | Agent handoff |
| **Model Required** | No | Yes | No |
| **Complexity** | Simple | Medium | High |
| **Cost** | Lowest | Medium | Low |
| **Predictability** | High | Medium | Low |

---

## Next Steps

- [14-swarm.md](14-swarm.md) - Agent-autonomous handoffs
- [15-termination-cancellation.md](15-termination-cancellation.md) - Control execution
- [28-troubleshooting.md](28-troubleshooting.md) - Debug infinite loops

---

## References

- See [11-teams.md](11-teams.md) for team fundamentals
- See [12-round-robin.md](12-round-robin.md) for fixed-order teams
- Official AutoGen: https://microsoft.github.io/autogen/
