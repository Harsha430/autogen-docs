# Part 20: Human-in-the-Loop Workflows

## What It Is

**HITL** integrates human decision-making into autonomous agent workflows.

```
Agent A produces output
  ↓
[PAUSE] Human approves/rejects/revises
  ↓
Continue OR restart
```

Agents make decisions, humans verify correctness.

---

## When to Use

✅ High-stakes decisions (financial, medical, legal)  
✅ Need human accountability ("Who approved this?")  
✅ Agent output needs review before action  
✅ Compliance requirements  
✅ Learning: agent ← human feedback  

❌ Fully autonomous systems (no human time)  
❌ Real-time constraints (humans too slow)  
❌ Decisions with no ambiguity  

---

## HITL Workflow Patterns

### Pattern 1: Sequential Approval

```
Agent → Produce → [Human Approval] → Execute
```

### Pattern 2: Revision Loop

```
Agent → Produce
  ↓
[Human Review]
  ├→ "Good" → Execute
  └→ "Revise" → Agent revises → Human reviews again
```

### Pattern 3: Expert Review

```
Agent A → Produce → [Human Expert Reviews] → Agent B executes
```

### Pattern 4: Escalation

```
Agent handles common cases
  ↓
Difficult case → [Human decides] → Direct response OR escalate to higher agent
```

---

## UserProxyAgent: The Human Proxy

`UserProxyAgent` represents a human in the team.

```python
from autogen_agentchat.agents import UserProxyAgent

human = UserProxyAgent(
    name="human_reviewer",
    input_func=input,  # Python's built-in input()
)

team = RoundRobinGroupChat(
    participants=[agent_a, human, agent_b],
    max_turns=10
)

result = await team.run(task="Review and approve report")
```

### When Human Speaks

```
Turn 1: agent_a produces output
Turn 2: human_reviewer
  Blocks execution
  Waits for human input at terminal:
  >>> [your input here]
  Sends human's response as TextMessage
Turn 3: agent_b continues with human's response
```

### Constructor

```python
UserProxyAgent(
    name: str,
    input_func: Callable = input,  # How to get human input
)
```

---

## Building an Approval Checkpoint

### Simple Approval Workflow

```python
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.teams import RoundRobinGroupChat

# Agent produces recommendation
recommender = AssistantAgent(
    name="recommender",
    model_client=model_client,
    system_message="Create a report with recommendation."
)

# Human approves
approver = UserProxyAgent(
    name="approver",
    input_func=input
)

# Execute report
executor = AssistantAgent(
    name="executor",
    model_client=model_client,
    system_message="Execute the approved plan."
)

team = RoundRobinGroupChat(
    participants=[recommender, approver, executor],
    max_turns=9  # 3 agents × 3 turns
)

result = await team.run(task="Propose marketing strategy")
```

### Execution Flow

```
Turn 1: recommender
  Output: "Proposed budget: $50K for social media"
  [Displayed to user]

Turn 2: approver (UserProxyAgent)
  >>> Do you approve? (yes/no/revise)
  Human types: "yes"
  [Sends: "Approved" as response]

Turn 3: executor
  "Thank you. Executing plan..."
  [Allocates budget, begins campaign]
```

---

## Approval with Revision Loop

### Pattern: "Revise Until Approved"

```python
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.termination import StopMessageTermination

# Producer
writer = AssistantAgent(
    name="writer",
    model_client=model_client,
    tools=[generate_content_tool],
    system_message="Create engaging product descriptions."
)

# Reviewer (human)
reviewer = UserProxyAgent(
    name="reviewer",
    input_func=input
)

team = RoundRobinGroupChat(
    participants=[writer, reviewer],
    max_turns=20,  # Allow multiple revisions
    termination_condition=StopMessageTermination(
        stop_words=["APPROVED", "FINAL", "PUBLISHED"]
    )
)

result = await team.run(task="Write product descriptions")
```

### Execution with Revision

```
Turn 1: writer
  Output: "Introducing the XYZ 2000..."

Turn 2: reviewer (UserProxyAgent)
  >>> Review feedback
  Human: "Too generic. Make it more exciting."

Turn 3: writer
  "Making it more energetic..."
  Output: "Revolutionize your workflow with XYZ 2000..."

Turn 4: reviewer
  >>> Review feedback
  Human: "Better! One typo: 'workflwo'. Also check price."

Turn 5: writer
  "Fixing typo and adding price..."
  Output: [corrected version]

Turn 6: reviewer
  >>> Review feedback
  Human: "APPROVED"

→ StopMessageTermination triggered → Execution stops
```

---

## State Tracking Across Checkpoints

HITL workflows often need to persist state between approval cycles.

### Problem: Approval Interrupts Workflow

```python
# This PAUSES execution at each human checkpoint
result = await team.run(...)
# ↑ Blocks waiting for human input

# If interrupted (Ctrl+C), state is lost
```

### Solution: Save State Before Approval

```python
from autogen_agentchat.agents import AssistantAgent

# After agent produces output, save state
agent = AssistantAgent(name="worker", model_client=client)

# Run until checkpoint
result = await agent.run(task="Generate report")

# Save state before human approval
agent.save_state("checkpoint_1.json")

# [Human reviews and approves]
# [Later, reload]
state = agent.load_state("checkpoint_1.json")
```

### Checkpoint in Team

```python
from autogen_agentchat.teams import RoundRobinGroupChat

team = RoundRobinGroupChat(
    participants=[producer, reviewer, executor],
    max_turns=10
)

# Run team
result = await team.run(task="...")

# After producer speaks but before executor:
# Save team state
import json
with open("team_checkpoint.json", "w") as f:
    json.dump({
        "messages": [msg.model_dump() for msg in result.messages],
        "last_speaker": result.messages[-1].source
    }, f)

# [Human reviews]
# [Later, resume from checkpoint]
```

---

## Common HITL Patterns

### Pattern 1: Content Moderation

```python
generator = AssistantAgent(name="generator", ...)
moderator = UserProxyAgent(name="moderator", input_func=input)

team = RoundRobinGroupChat(
    participants=[generator, moderator]
)

# Turn 1: generator creates content
# Turn 2: moderator (human) approves or flags
```

### Pattern 2: Expense Approval

```python
employee = AssistantAgent(
    name="expense_requester",
    system_message="Fill expense form with details"
)

manager = UserProxyAgent(
    name="manager_approval",
    input_func=input
)

team = RoundRobinGroupChat(participants=[employee, manager])
# Manager approves or requests receipt
```

### Pattern 3: Research Review

```python
researcher = AssistantAgent(name="researcher", ...)
fact_checker = AssistantAgent(name="fact_checker", ...)
human_expert = UserProxyAgent(name="human_expert", input_func=input)

team = Swarm(
    participants=[researcher, fact_checker, human_expert]
)
# Researcher → Fact checker → Human expert for final sign-off
```

### Pattern 4: Code Review

```python
coder = AssistantAgent(name="coder", ...)
tester = AssistantAgent(name="tester", ...)
senior_reviewer = UserProxyAgent(name="senior_reviewer", input_func=input)

team = SelectorGroupChat(
    participants=[coder, tester, senior_reviewer],
    selector_prompt="Who reviews next?"
)
# Selector routes when human review is needed
```

---

## Programmatic Approval (Without Terminal Input)

Not all HITL needs interactive terminal input. Some workflows use APIs, emails, or web forms.

### Using a Queue System

```python
import asyncio

class QueuedApprover:
    """Approves via message queue (simulates human approval)"""
    
    def __init__(self):
        self.pending = []  # Queue of things needing approval
    
    def get_human_input(self, message: str) -> str:
        """Get approval from queue or return pending"""
        # In real system: check HTTP API, email, etc.
        self.pending.append(message)
        print(f"Added to approval queue: {message[:50]}...")
        return "APPROVED"  # Simulate human approval

approver = QueuedApprover()

human = UserProxyAgent(
    name="approver",
    input_func=approver.get_human_input
)

team = RoundRobinGroupChat(
    participants=[agent, human]
)

result = await team.run(task="...")
print(f"Approval queue: {approver.pending}")
```

### Using Webhooks

```python
import requests

def webhook_approval(task_desc: str) -> str:
    """Send to webhook, wait for response"""
    
    response = requests.post(
        "https://approval-service.internal/submit",
        json={"task": task_desc}
    )
    
    if response.status_code == 200:
        return "APPROVED"
    else:
        return "REJECTED: Please review manually"

human = UserProxyAgent(
    name="approval_webhook",
    input_func=webhook_approval
)
```

---

## Cost Considerations

HITL workflows have real costs: human time and attention.

### Cost Analysis

```python
# Scenario: 100 documents to approve
# Each takes human 2 minutes

cost_human_time = 100 * 2 / 60  # 3.3 hours
cost_human_salary = 3.3 * ($80/hour) = ~$264

# vs. Full automation (risky, no approval)
cost_full_auto = $0.50 (API calls)

# vs. Hybrid (agent flags high-risk, human reviews)
high_risk = 10  # 10% flagged
cost_hybrid = 10 * 2 / 60 * $80 = ~$27
```

### When HITL ROI Makes Sense

✅ High-stakes decisions (financial loss >> human time)  
✅ Legal/compliance requirements  
✅ Reputational risk  

❌ Repetitive, low-stakes approvals (too expensive)  
❌ Time-sensitive (humans too slow)  

### Optimization: Risk-Based HITL

```python
# Only human-review high-risk decisions
from autogen_agentchat.messages import TextMessage

def should_need_approval(output: str) -> bool:
    """Check if output warrants human review"""
    risk_factors = [
        len(output) > 1000,  # Long recommendations
        "delete" in output.lower(),  # Destructive
        "financial" in output.lower(),  # Money involved
    ]
    return sum(risk_factors) >= 2

# In workflow
agent_output = await agent.run(...)

if should_need_approval(agent_output):
    human_response = input("Approve? (yes/no): ")
else:
    human_response = "AUTO_APPROVED"
```

---

## Common Mistakes

### Mistake 1: Blocking User Forever
```python
# WRONG - No timeout
human = UserProxyAgent(
    name="reviewer",
    input_func=input  # Blocks indefinitely if user doesn't respond
)

# RIGHT - Use timeout or async input
import signal

def input_with_timeout(prompt, timeout=300):  # 5 minutes
    """Timeout human input after 5 minutes"""
    # Implementation depends on OS
    pass
```

### Mistake 2: No State Across Runs
```python
# WRONG - Restarts with no memory
result1 = await team.run(...)
# [Human approves]
result2 = await team.run(...)  # Starts from scratch!

# RIGHT - Load previous state
team.load_state("previous_checkpoint.json")
result2 = await team.run(...)
```

### Mistake 3: HITL at Every Step
```python
# WRONG - Too many approval points (human exhausted)
participants = [agent1, human, agent2, human, agent3, human]
# User approves 6 times for one task!

# RIGHT - Approve at key checkpoints only
participants = [agent1, agent2, agent3, human_final_review]
```

### Mistake 4: Ambiguous Approval Message
```python
# WRONG - Human confused
>>> Approve? (yes/no)

# RIGHT - Clear options
>>> Review output above.
>>> Options: [A]pprove, [R]eject, [M]odify, [C]ancel
```

---

## Debugging HITL Workflows

### Check Approval Sequence
```python
result = await team.run(...)

for i, msg in enumerate(result.messages):
    print(f"Turn {i+1}: {msg.source}")
    if msg.source == "approver":
        print(f"  Human said: {msg.content}")
```

### Trace Handoffs
```python
from autogen_agentchat.messages import HandoffMessage

for msg in result.messages:
    if isinstance(msg, HandoffMessage):
        print(f"Handoff: {msg.source} → {msg.to}")
```

### Check Approval Rate
```python
approvals = [m for m in result.messages if "APPROVED" in m.content]
rejections = [m for m in result.messages if "REJECTED" in m.content]
revisions = [m for m in result.messages if "REVISE" in m.content]

print(f"Approved: {len(approvals)}, Rejected: {len(rejections)}, Revisions: {len(revisions)}")
```

---

## Comparison: HITL vs Pure Automation

| Aspect | HITL | Pure Automation |
|--------|------|-----------------|
| **Speed** | Slow (human delay) | Fast |
| **Accuracy** | High (human oversight) | Variable |
| **Accountability** | Clear ("human approved") | Ambiguous |
| **Cost** | High (human time) | Low |
| **Scalability** | Poor (human bottleneck) | High |
| **Risk** | Managed | Higher |

---

## Next Steps

- [15-termination-cancellation.md](15-termination-cancellation.md) - Handle interruptions
- [21-observability.md](21-observability.md) - Track approval workflow
- [25-complete-project.md](25-complete-project.md) - Full example with HITL

---

## References

- See [06-user-proxy-agent.md](06-user-proxy-agent.md) for UserProxyAgent details
- See [11-teams.md](11-teams.md) for team fundamentals
- Official AutoGen: https://microsoft.github.io/autogen/
