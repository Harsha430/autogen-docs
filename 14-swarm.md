# Part 14: Swarm - Handoff-Based Coordination

## What It Is

**Swarm** lets agents autonomously transfer control to other agents using **HandoffMessage**.

```
Agent A handles request
  ↓ (decides it needs help)
"I'll handoff to Agent B"
  ↓ (sends HandoffMessage)
Agent B handles request
  ↓ (decides to handoff)
"I'll handoff to Agent C"
  ↓
Agent C handles and returns result
```

Agents decide when to handoff—no external orchestrator needed.

---

## When to Use

✅ Agents naturally defer to specialists  
✅ Chain of command workflows  
✅ Escalation patterns  
✅ Agents have explicit domain boundaries  
✅ Autonomous agent networks  

❌ When order depends on content (use Selector)  
❌ Simple sequential tasks (use RoundRobin)  
❌ When humans need to choose (use HITL)  

---

## Constructor

```python
from autogen_agentchat.teams import Swarm

team = Swarm(
    participants=[agent1, agent2, agent3],
    max_turns=20,
    termination_condition=None
)
```

### Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| `participants` | list[Agent] | All reachable agents |
| `max_turns` | int \| None | Max conversation turns |
| `termination_condition` | TerminationCondition \| None | Stop condition |

**Note**: No `model_client` needed (unlike Selector) because agents decide routing.

---

## HandoffMessage: The Core Mechanism

Agents communicate handoffs using `HandoffMessage`:

```python
from autogen_agentchat.messages import HandoffMessage

# Inside an agent's response
handoff = HandoffMessage(
    to="specialist_agent",
    content="Please handle this specialized task"
)
return [handoff]
```

### Structure

```python
HandoffMessage(
    to: str,              # Target agent name
    content: str,         # Why/what to hand off
    source: str = "...",  # Sending agent (auto-filled)
)
```

---

## How Swarm Works

### Phase 1: Agent Processes
Current agent receives task, analyzes it, may call tools.

### Phase 2: Agent Decides
- **Option A**: "I can handle this" → Produce response
- **Option B**: "I need help" → Generate `HandoffMessage`

### Phase 3: Swarm Routes
If `HandoffMessage` received:
- Extract target agent name from `to`
- Find agent with matching name
- Transfer control

### Phase 4: Next Agent Processes
New agent receives the handoff message and context.

```
Turn 1: agent_a
  Analyzes request
  Decides: "This needs a specialist"
  Sends: HandoffMessage(to="specialist_b", content="...")

Turn 2: specialist_b
  Receives handoff + context
  Processes task
  Decides: "Complete" OR "Handoff to specialist_c"

Turn 3: specialist_c (if handoff received)
  Continues work
  Returns final result

→ Task complete
```

---

## Agent Handoff Configuration

Each agent must know WHO it can handoff to.

### Method: System Message

```python
agent_a = AssistantAgent(
    name="agent_a",
    system_message="""
    You are a request triage agent.
    - If it's a billing question: handoff to billing_specialist
    - If it's technical: handoff to technical_specialist
    - If it's a complaint: handoff to customer_care_specialist
    
    Use HandoffMessage to transfer:
    from autogen_agentchat.messages import HandoffMessage
    return [HandoffMessage(to="billing_specialist", content="...")]
    """
)

specialist_a = AssistantAgent(
    name="billing_specialist",
    system_message="""
    You are a billing expert. Handle billing questions.
    If this requires legal review, handoff:
    HandoffMessage(to="legal_specialist", content="...")
    """
)
```

---

## Routing Logic: Important Distinction

### Swarm Routing ≠ Selector Routing

**Selector**:
- Central orchestrator
- Model decides: "A should speak now"
- External routing logic

**Swarm**:
- Distributed decision-making
- Agent decides: "I should handoff to B"
- Internal routing logic in agent

```python
# SELECTOR: Centralized
selector_prompt = "Who should speak next?"
# Model: "Agent B"

# SWARM: Distributed
# Agent's logic: "I should handoff to B"
# Agent produces: HandoffMessage(to="B", ...)
```

### When to Choose

| Pattern | Use This |
|---------|----------|
| "Coordinator picks specialist" | Selector |
| "Specialist says 'get me another specialist'" | Swarm |
| "Job routing system" | Selector |
| "Customer service escalation" | Swarm |

---

## Example: Customer Support Escalation

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import Swarm
from autogen_agentchat.messages import HandoffMessage

# Level 1: Front-line support
frontline = AssistantAgent(
    name="frontline_support",
    model_client=model_client,
    system_message="""
    You are frontline customer support. Resolve simple issues:
    - Password reset → Do it
    - Account info → Provide
    - Complex issues → Handoff to specialist
    
    For complex issues, use:
    from autogen_agentchat.messages import HandoffMessage
    return [HandoffMessage(
        to="billing_specialist",
        content="Customer needs billing help: ..."
    )]
    """
)

# Level 2: Billing specialist
billing = AssistantAgent(
    name="billing_specialist",
    model_client=model_client,
    system_message="""
    You are billing expert. Handle:
    - Refunds
    - Subscription changes
    - Payment issues
    
    If legal issue, handoff:
    HandoffMessage(to="legal_specialist", content="...")
    """
)

# Level 3: Legal specialist
legal = AssistantAgent(
    name="legal_specialist",
    model_client=model_client,
    system_message="You are legal expert. Handle disputes and contracts."
)

# Create swarm
team = Swarm(
    participants=[frontline, billing, legal],
    max_turns=10
)

# Run customer query
result = await team.run(
    task="Customer wants refund for subscription they claim was fraudulent"
)
```

### Execution Trace

```
Turn 1: frontline_support
  "This is a complex billing/legal issue. Let me escalate."
  HandoffMessage(to="billing_specialist", ...)

Turn 2: billing_specialist
  "I see a refund request with fraud claim. This needs legal."
  HandoffMessage(to="legal_specialist", ...)

Turn 3: legal_specialist
  "Based on terms of service..."
  Produces final decision
  
→ Result returned
```

---

## Infinite Loop Prevention in Swarm

More critical than Selector because agents control routing.

### Problem: Circular Handoffs
```
Agent A → Agent B → Agent A → Agent B → ...
```

### Problem: Dead Ends
```
Agent A handoffs to unknown agent name
"HandoffMessage(to="unknown_agent", ...)"
→ Agent not found error
```

### Solutions

**1. Explicit Handoff Graph**
```python
# Document who can handoff to whom
handoff_graph = {
    "frontline": ["billing", "technical"],     # Frontline → 2 options
    "billing": ["legal"],                       # Billing → Legal only
    "legal": [],                                # Legal → nowhere
    "technical": ["senior_engineer"],
    "senior_engineer": []
}

# Before handoff, verify:
if target in handoff_graph[current_agent]:
    # Valid handoff
else:
    # Invalid - raise error or handle
```

**2. Terminal Agents**
```python
# Some agents are "final" - they don't handoff
legal_agent = AssistantAgent(
    name="legal_specialist",
    system_message="""
    You are the final authority. Make a decision and STOP.
    Never handoff - your decision is final.
    """
)
```

**3. Breadcrumb Tracking**
```python
# Track handoff path
path = []
for msg in result.messages:
    if isinstance(msg, HandoffMessage):
        path.append((msg.source, msg.to))

# Detect: A→B→A
for i, (from_agent, to_agent) in enumerate(path):
    if i > 0 and to_agent == path[i-1][0]:
        print(f"Loop detected: {from_agent}→{to_agent}→{from_agent}")
```

**4. Strict max_turns**
```python
team = Swarm(
    participants=[...],
    max_turns=5,  # Hard limit
    termination_condition=MaxMessageTermination(max_messages=30)
)
```

---

## Common Mistakes

### Mistake 1: Invalid Agent Name in HandoffMessage
```python
# WRONG - typo in agent name
return [HandoffMessage(to="billing_specailist", ...)]
# ↑ "specailist" doesn't exist!

# RIGHT - verify name matches actual agent
team = Swarm(participants=[billing_specialist])  # name="billing_specialist"
return [HandoffMessage(to="billing_specialist", ...)]
```

### Mistake 2: Agents Don't Know Who to Handoff To
```python
# WRONG - No guidance
agent = AssistantAgent(
    name="a",
    system_message="Handle requests."  # Doesn't know handoff targets
)

# RIGHT - Explicit
agent = AssistantAgent(
    name="a",
    system_message="""
    Handle requests. For billing: handoff to billing_specialist.
    For technical: handoff to technical_specialist.
    Use: HandoffMessage(to="...", content="...")
    """
)
```

### Mistake 3: Missing HandoffMessage Import
```python
# WRONG - Crashes at runtime
system_message = """
Return HandoffMessage(to="other_agent", ...)
"""  # HandoffMessage not imported in context

# RIGHT - Clear import in agent context or examples
from autogen_agentchat.messages import HandoffMessage
```

### Mistake 4: Circular Handoff Graph
```python
# WRONG - A↔B loop
a.system_message = "If complex, handoff to B"
b.system_message = "If needs first review, handoff to A"

# RIGHT - Directed graph (no cycles)
# A → B → C (not B → A)
```

---

## Output Inspection

```python
result = await team.run(task="...")

# See handoff path
handoffs = [msg for msg in result.messages 
            if isinstance(msg, HandoffMessage)]

print(f"Total handoffs: {len(handoffs)}")
for handoff in handoffs:
    print(f"{handoff.source} → {handoff.to}")

# See agent sequence
agents = [msg.source for msg in result.messages]
print(f"Agent sequence: {' → '.join(agents)}")

# Final result
print(f"Final response: {result.messages[-1]}")
```

---

## Swarm vs Selector: When to Choose

| Aspect | Swarm | Selector |
|--------|-------|----------|
| **Who decides routing?** | Agents | Central model |
| **Routing logic** | In agent system message | In selector_prompt |
| **Agent autonomy** | High (agents choose) | Low (told where to go) |
| **Overhead** | Low (no extra model calls) | Medium (model calls per turn) |
| **Debugging difficulty** | Harder (distributed logic) | Easier (centralized) |
| **Natural fit** | "Escalation chains" | "Smart coordinator" |

**Example mapping**:
- Customer service escalation → Swarm
- Code review with smart routing → Selector
- Job dispatch with agent decision → Swarm
- Content routing to expert → Selector

---

## Advanced: Conditional Handoffs

Agents can use tools to decide whether to handoff:

```python
agent = AssistantAgent(
    name="analyzer",
    tools=[assess_complexity_tool],
    system_message="""
    Analyze the request.
    - Call assess_complexity_tool
    - If complexity > 7: handoff to expert
    - Otherwise: handle yourself
    
    from autogen_agentchat.messages import HandoffMessage
    if complexity > 7:
        return [HandoffMessage(to="expert", content=f"Complexity: {complexity}")]
    else:
        return [TextMessage(content="Handled")]
    """
)
```

---

## Next Steps

- [13-selector-group-chat.md](13-selector-group-chat.md) - Compare centralized routing
- [15-termination-cancellation.md](15-termination-cancellation.md) - Control execution
- [20-human-in-loop.md](20-human-in-loop.md) - Add human approval

---

## References

- See [11-teams.md](11-teams.md) for team fundamentals
- See [07-messages.md](07-messages.md) for HandoffMessage details
- Official AutoGen: https://microsoft.github.io/autogen/
