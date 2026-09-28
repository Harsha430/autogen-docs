# Part 6: UserProxyAgent - Human in the Team

## What It Is

**UserProxyAgent** represents a human in a team of agents. It blocks execution waiting for human input.

```python
from autogen_agentchat.agents import UserProxyAgent

human = UserProxyAgent(
    name="human_reviewer",
    input_func=input  # Gets human input from terminal
)

team = RoundRobinGroupChat(
    participants=[agent_a, human, agent_b]
)
result = await team.run(task="Review proposal")

# Execution pauses when human's turn comes
# >>> [User enters text]
# Human's input becomes a message in the conversation
```

---

## When to Use

✅ Need human approval in workflow  
✅ Human should make decisions  
✅ Interactive approval loops  
✅ Testing (simulate human responses)  

❌ Fully autonomous systems (no human time)  
❌ Want human input but not in team (use run_stream())  
❌ Human can't respond during execution (use HITL polling)  

---

## Constructor

```python
UserProxyAgent(
    name: str,
    input_func: Callable[[str], str] = input,
)
```

### Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| `name` | str | Agent name (e.g., "human_reviewer") |
| `input_func` | Callable | Function that gets human input |

---

## input_func: Customizing Input

### Built-in: Terminal Input

```python
# Uses Python's input() function
human = UserProxyAgent(
    name="reviewer",
    input_func=input  # Blocks at terminal
)

# When human's turn comes:
# >>> [system prompts with agent's last message]
# >>> [User types response and presses Enter]
```

### Custom: With Prompt

```python
def get_approval(prompt: str) -> str:
    """Custom input with better prompt"""
    print("\n" + "="*50)
    print(prompt)
    print("="*50)
    user_input = input("Your response: ").strip()
    return user_input

human = UserProxyAgent(
    name="approver",
    input_func=get_approval
)
```

### Custom: Predefined Responses (Testing)

```python
def mock_human_responses(prompt: str) -> str:
    """Simulate human for testing"""
    if "approve" in prompt.lower():
        return "APPROVED"
    elif "revise" in prompt.lower():
        return "Revise for clarity"
    else:
        return "Continue"

human = UserProxyAgent(
    name="test_human",
    input_func=mock_human_responses
)

# Now team.run() doesn't block - uses predefined responses
result = await team.run(task="...")
print(result.messages)  # Includes "APPROVED", etc.
```

### Custom: Webhook/API

```python
import requests

def webhook_approval(prompt: str) -> str:
    """Send to approval service, get response"""
    response = requests.post(
        "https://approval-service.example.com/approve",
        json={"prompt": prompt, "task": "content_review"}
    )
    if response.status_code == 200:
        return response.json()["decision"]
    else:
        return "REJECTED: Service unavailable"

human = UserProxyAgent(
    name="api_approver",
    input_func=webhook_approval
)
```

---

## Usage Pattern

```python
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.teams import RoundRobinGroupChat

# Create AI agent
ai_agent = AssistantAgent(
    name="writer",
    model_client=model_client,
    system_message="Write content based on requirements"
)

# Create human proxy
human = UserProxyAgent(
    name="editor",
    input_func=input
)

# Create team
team = RoundRobinGroupChat(
    participants=[ai_agent, human],
    max_turns=6  # 3 rounds: AI → Human → AI → Human → ...
)

# Run
result = await team.run(task="Write blog post about Python")
```

### Execution with Interactive Input

```
Turn 1: writer
  "I'll write a blog post about Python..."
  [Produces: "Python is a powerful language..."]

Turn 2: editor (UserProxyAgent - PAUSES HERE)
  >>> Please review the blog post above.
  >>> Options: [A]pproved, [R]evise, [C]ancel
  >>> (Your response: )
  
  [User types at terminal: "Revise for more examples"]

Turn 3: writer
  "I'll add more examples..."
  [Produces: revised version]

Turn 4: editor (Pauses again)
  >>> Review revised content
  >>> (Your response: )
  
  [User types: "Approved"]

→ Execution continues or terminates based on stop condition
```

---

## Difference: UserProxyAgent vs HITL

| Aspect | UserProxyAgent | HITL |
|--------|---|---|
| **How** | Blocks in team | Pauses workflow, resumes later |
| **When** | During execution | Before/after stages |
| **Use** | Interactive real-time | Asynchronous review |
| **Example** | Chat in terminal | Email approval then resume |

**UserProxyAgent**: Human MUST be present during execution

**HITL**: Human reviews at their convenience, workflow resumes

---

## Example 1: Interactive Code Review

```python
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.teams import RoundRobinGroupChat

developer = AssistantAgent(
    name="developer",
    model_client=model_client,
    system_message="Write Python code based on requirements"
)

human_reviewer = UserProxyAgent(
    name="code_reviewer",
    input_func=input
)

team = RoundRobinGroupChat(
    participants=[developer, human_reviewer],
    max_turns=8
)

result = await team.run(
    task="Write a Python function to validate email addresses"
)

# Flow:
# Dev: Writes function
# Human: Reviews code, asks for changes
# Dev: Revises based on feedback
# Human: Approves
```

---

## Example 2: Testing with Mock Responses

```python
def mock_responses(prompt: str) -> str:
    """Predefined test responses"""
    responses = {
        "approve": "APPROVED - looks good",
        "reject": "REJECTED - needs fixes",
        "revise": "REVISE - add comments",
    }
    
    for key, value in responses.items():
        if key in prompt.lower():
            return value
    
    return "CONTINUE"

# In test
human = UserProxyAgent(
    name="test_reviewer",
    input_func=mock_responses
)

team = RoundRobinGroupChat(
    participants=[agent, human],
    max_turns=4
)

result = await team.run(task="Generate report")

# Completes without user input (uses mock responses)
assert any("APPROVED" in msg.content for msg in result.messages)
```

---

## Example 3: Conditional Approval

```python
def conditional_approval(prompt: str) -> str:
    """Approve based on conditions"""
    
    # Parse what needs approval
    if "budget" in prompt.lower() and "$" in prompt:
        # Extract amount
        import re
        match = re.search(r'\$(\d+)', prompt)
        if match:
            amount = int(match.group(1))
            if amount < 1000:
                return "APPROVED - under 1K threshold"
            else:
                return "REJECTED - over budget limit"
    
    # Default ask user
    print(prompt)
    return input("Your decision: ")

human = UserProxyAgent(
    name="smart_approver",
    input_func=conditional_approval
)
```

---

## Limitations

### 1. Blocking During Execution
```python
# UserProxyAgent BLOCKS and waits
result = await team.run(...)  # Pauses if human's turn
# You MUST be present to type response
```

### 2. Can't Handle Async Input
```python
# WRONG - Won't work with async human
async def async_approval(prompt: str) -> str:
    # This is sync, can't use await inside
    await some_async_operation()  # ✗ Fails

# RIGHT - Use sync operations
def sync_approval(prompt: str) -> str:
    # Only sync operations
    return input(prompt)
```

### 3. No Timeout
```python
# WRONG - If human never responds, blocks forever
human = UserProxyAgent(name="approver", input_func=input)

# RIGHT - Add timeout in input_func
import signal

def input_with_timeout(prompt: str, timeout=300):
    def timeout_handler(signum, frame):
        raise TimeoutError("Input timeout")
    
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(timeout)
    try:
        result = input(prompt)
        signal.alarm(0)  # Cancel alarm
        return result
    except TimeoutError:
        return "TIMEOUT - skipping"
```

---

## Common Mistakes

### Mistake 1: Forgetting It Blocks
```python
# WRONG - Assumes no blocking
result = await team.run(...)  # Blocks at human's turn!
print("Done!")  # Never reached until human responds

# RIGHT - Account for blocking
result = await team.run(...)  # Blocks here
# User provides input
print("Done!")  # Reached after human responds
```

### Mistake 2: Wrong input_func Signature
```python
# WRONG - input_func takes no arguments
def get_response():  # No parameter!
    return input(">> ")

human = UserProxyAgent(name="h", input_func=get_response)

# RIGHT - input_func takes prompt string
def get_response(prompt: str) -> str:  # Takes prompt!
    return input(prompt)

human = UserProxyAgent(name="h", input_func=get_response)
```

### Mistake 3: input_func Doesn't Return String
```python
# WRONG - Returns int
def get_response(prompt: str) -> str:
    return 42  # ✗ Should be string

# RIGHT - Returns string
def get_response(prompt: str) -> str:
    return input(prompt)  # ✓ String
```

### Mistake 4: Using in Fully Autonomous Context
```python
# WRONG - Expects no human but blocks waiting
async def autonomous_system():
    human = UserProxyAgent(name="h", input_func=input)
    team = SelectorGroupChat(
        participants=[agent_a, agent_b, human]
    )
    result = await team.run(...)  # Blocks if human selected!

# RIGHT - No human in autonomous system
async def autonomous_system():
    team = SelectorGroupChat(
        participants=[agent_a, agent_b]  # No human
    )
    result = await team.run(...)  # Fully autonomous
```

---

## When to Use UserProxyAgent vs Alternatives

| Need | Use | Why |
|------|-----|-----|
| Interactive approval | UserProxyAgent | Blocks for real-time |
| Asynchronous review | HITL + save_state | Resume later |
| Testing | UserProxyAgent + mock | No blocking |
| Fully autonomous | No UserProxyAgent | Can't block |

---

## Next Steps

- [20-human-in-loop.md](20-human-in-loop.md) - Asynchronous human workflows
- [07-messages.md](07-messages.md) - Message types
- [11-teams.md](11-teams.md) - Teams with humans

---

## References

- See [04-assistant-agent.md](04-assistant-agent.md) for AI agents
- See [05-custom-agents.md](05-custom-agents.md) for custom agents
- Official AutoGen: https://microsoft.github.io/autogen/
