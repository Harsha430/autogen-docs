# Part 28: Troubleshooting - Common Issues & Solutions

## Quick Diagnosis Flowchart

```
Issue occurs
  ├─→ No output? → See: Agents Not Responding
  ├─→ Error message? → See: Common Error Messages
  ├─→ Too slow? → See: Performance Degradation
  ├─→ Loop forever? → See: Infinite Loops
  ├─→ Tool doesn't work? → See: Tool Execution Failures
  ├─→ State file won't load? → See: State Loading Errors
  └─→ Memory usage climbing? → See: Memory Leaks
```

---

## Issue 1: "Model not found" Error

### Error Message
```
ValueError: Model 'gpt-4' not found in available models
```

### Why This Happens

- Model name is misspelled
- Model not available in your API plan
- API account doesn't have access
- Wrong API key for model family

### Quick Fixes

```python
# WRONG
client = AzureOpenAIClient(
    model="gpt-4"  # ← Wrong name or not available
)

# RIGHT - Check available models
# OpenAI: "gpt-4", "gpt-4-turbo", "gpt-4o"
# Azure: "gpt-4-deployment-name" (exact deployment name)
# Anthropic: "claude-3-5-sonnet-20241022"

client = OpenAIClient(
    model="gpt-4-turbo",
    api_key="sk-..."
)
```

### Debug Steps

**Step 1**: List available models
```python
from autogen_agentchat import list_models
print(list_models())  # Shows all configured models
```

**Step 2**: Verify API key
```python
# Check if API key is set
import os
print(os.getenv("OPENAI_API_KEY"))  # Should not be None
```

**Step 3**: Check pricing tier
- OpenAI: "gpt-4" limited to paid accounts
- Azure: Deployment must exist
- Anthropic: Access via API key grants all models

### Correct Mental Model

Models aren't "found" — they're **requested from the provider's API**.
- Your API key must grant access
- Model name must match provider's naming
- Region/deployment availability matters for cloud providers

---

## Issue 2: API Key Errors

### Error Messages
```
AuthenticationError: Unauthorized
Invalid API key
401 Unauthorized
```

### Why This Happens

- API key not set
- API key invalid/expired
- API key has wrong permissions
- Wrong API key for provider

### Quick Fixes

```python
# WRONG - Hardcoded (exposed in repo!)
client = OpenAIClient(api_key="sk-proj-xxx")

# WRONG - Environment variable not set
import os
api_key = os.getenv("OPENAI_API_KEY")  # Returns None!

# RIGHT - Environment variable properly set
# Before running:
# export OPENAI_API_KEY="sk-proj-xxx"
# (or set in .env file)

import os
from dotenv import load_dotenv
load_dotenv()

client = OpenAIClient(
    api_key=os.getenv("OPENAI_API_KEY")
)
```

### Debug Steps

**Step 1**: Verify environment variable
```python
import os
key = os.getenv("OPENAI_API_KEY")
if not key:
    print("ERROR: OPENAI_API_KEY not set!")
    print("Set it with: export OPENAI_API_KEY='sk-...'")
else:
    print(f"API key found: {key[:10]}...")  # Show first 10 chars
```

**Step 2**: Test API connection
```python
from openai import OpenAI

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

try:
    response = client.models.list()
    print(f"✓ Connected! Found {len(response.data)} models")
except Exception as e:
    print(f"✗ Connection failed: {e}")
```

**Step 3**: Regenerate key if needed
- OpenAI: https://platform.openai.com/account/api-keys
- Azure: Azure Portal → OpenAI resource → Keys
- Anthropic: https://console.anthropic.com/

### Correct Mental Model

API keys are **authentication credentials** — they prove you paid and have access.
- Never commit them to git
- Rotate them periodically
- Different services need different keys
- Expiration depends on provider policy

---

## Issue 3: Token Limit Exceeded

### Error Message
```
RateLimitError: Rate limit exceeded
BadRequestError: This model's maximum context length is 8192 tokens
```

### Why This Happens

- Conversation history too long
- Single prompt too long
- Too many tool calls adding to context
- Model change (new model has lower limit)

### Quick Fixes

```python
# WRONG - Unlimited history
team = RoundRobinGroupChat(
    participants=[agent1, agent2],
    max_turns=None  # ← Can run forever!
)

# RIGHT - Bounded turns
team = RoundRobinGroupChat(
    participants=[agent1, agent2],
    max_turns=20,  # ~50-100 messages depending on length
    termination_condition=MaxMessageTermination(max_messages=50)
)
```

### Token Management

```python
from autogen_agentchat.model_client import token_count

# Check message size
message = "Your very long conversation history..."
tokens = token_count(message)
print(f"Message: {tokens} tokens")

# For model with 8K limit:
# Reserve 1K for response = 7K available
# If conversation already uses 6K, only 1K left for new prompt
```

### Strategies

**Strategy 1: Reduce Conversation Turns**
```python
# WRONG - 20 turns = 200-400 messages
team = RoundRobinGroupChat(max_turns=20)

# RIGHT - Fewer turns, more per-turn work
team = RoundRobinGroupChat(max_turns=5)
```

**Strategy 2: Archive Old Messages**
```python
# Keep recent messages, summarize old ones
def keep_recent_messages(messages, keep_count=20):
    if len(messages) > keep_count:
        old_summary = f"[Previous {len(messages)-keep_count} messages summarized]"
        return [TextMessage(content=old_summary)] + messages[-keep_count:]
    return messages

recent = keep_recent_messages(result.messages)
```

**Strategy 3: Use Larger Model**
```python
# WRONG - Small model, limited context
client = OpenAIClient(model="gpt-3.5-turbo")  # 4K tokens

# RIGHT - Larger model, more context
client = OpenAIClient(model="gpt-4-turbo")  # 128K tokens
```

### Monitor Token Usage

```python
result = await agent.run(...)

# Tokens used (varies by provider)
print(f"Stop reason: {result.stop_reason}")

# Count messages
message_count = len(result.messages)
avg_tokens_per_message = 50  # Rough estimate
estimated_tokens = message_count * avg_tokens_per_message
print(f"~{estimated_tokens} tokens used")
```

### Correct Mental Model

Token limits are **hard constraints** — you cannot exceed them.
- Each model has different limits
- Every message (input and output) counts
- Tool calls and results add tokens
- Plan your conversation to stay well under limit

---

## Issue 4: Infinite Loops

### Symptoms

```
Turn 1: Agent A
Turn 2: Agent B
Turn 3: Agent A
Turn 4: Agent B
Turn 5: Agent A
... continues forever until max_turns or you Ctrl+C
```

### Why This Happens

- Agents keep responding to each other with no progress
- Selector keeps choosing same two agents
- Swarm handoff goes A→B→A→B...
- Termination condition never triggers

### Quick Fixes

```python
# WRONG - No termination
team = SelectorGroupChat(
    participants=[A, B],
    selector_prompt="Who should speak?",
    max_turns=None  # ← Will run forever!
)

# RIGHT - Always have max_turns + condition
team = SelectorGroupChat(
    participants=[A, B],
    max_turns=20,  # Hard limit
    termination_condition=StopMessageTermination(
        stop_words=["COMPLETE", "DONE"]
    )
)
```

### Detect Loops

```python
from collections import Counter

result = await team.run(...)
speakers = [msg.source for msg in result.messages]

# Count speaker frequency
counts = Counter(speakers)
print(f"Speaker counts: {counts}")

# Detect: Only 2 agents speaking
if len(counts) == 2 and result.stop_reason == "max_turns":
    print("⚠️ WARNING: Possible infinite loop!")
    print(f"Only {list(counts.keys())} spoke")
    
    # Check pattern
    pattern = [speakers[i] for i in range(min(10, len(speakers)))]
    print(f"First 10 turns: {pattern}")
```

### Prevent Loops: Selector

**Problem**: Selector keeps choosing A and B
```python
# WRONG - Too vague
selector_prompt = "Who should speak next?"

# RIGHT - Require progress
selector_prompt = """
Choose the next speaker, but:
1. Avoid repeating the same two agents
2. If stuck, escalate to a decision-maker
3. Move toward a conclusion
"""
```

### Prevent Loops: Swarm

**Problem**: A handoffs to B, B handoffs to A
```python
# WRONG - Circular
a_system = "If complex, handoff to B"
b_system = "If needs initial review, handoff to A"

# RIGHT - Directed flow (no cycles)
a_system = "Analyze. Handoff to B for implementation."
b_system = "Implement. Never handoff back to A."
```

### Prevent Loops: General

```python
# In any team, add stopping condition
team = SelectorGroupChat(
    participants=[...],
    termination_condition=StopMessageTermination(
        stop_words=["FINAL", "APPROVED", "COMPLETE"]
    )
)

# Agents must say these words to exit
agent_system = """
When you've finished, end with: "COMPLETE"
This signals the team to stop.
"""
```

### Correct Mental Model

Infinite loops happen because **agents have no exit condition**.
- Max_turns is safety net, not primary exit
- Termination conditions express "success"
- Agents should work toward termination
- Detect loops by watching speaker frequency

---

## Issue 5: Agents Not Responding

### Symptoms

```
[Minutes pass, nothing happens]
Or:
No error, but result.messages is empty
```

### Why This Happens

- Model client not initialized
- API call times out
- Agent has empty system_message
- Tool calling is broken
- Network connectivity issue

### Quick Fixes

```python
# WRONG - No model_client
agent = AssistantAgent(
    name="worker",
    model_client=None  # ← Will crash!
)

# RIGHT
from autogen_agentchat.clients import OpenAIClient
client = OpenAIClient(model="gpt-4-turbo", api_key="sk-...")

agent = AssistantAgent(
    name="worker",
    model_client=client
)
```

### Debug Steps

**Step 1**: Verify model_client works
```python
client = OpenAIClient(model="gpt-4-turbo")

# Test directly
try:
    response = client.create(
        messages=[{"role": "user", "content": "Say hello"}],
        temperature=0.7
    )
    print(f"✓ Client works: {response}")
except Exception as e:
    print(f"✗ Client failed: {e}")
```

**Step 2**: Verify agent creation
```python
agent = AssistantAgent(name="test", model_client=client)
print(f"✓ Agent created: {agent.name}")
```

**Step 3**: Run with timeout
```python
import asyncio

try:
    result = await asyncio.wait_for(
        agent.run(task="Hello"),
        timeout=30  # 30 second timeout
    )
    print(f"✓ Completed: {result.messages}")
except asyncio.TimeoutError:
    print("✗ Timed out - API may be slow")
except Exception as e:
    print(f"✗ Error: {e}")
```

**Step 4**: Enable debug logging
```python
import logging
logging.basicConfig(level=logging.DEBUG)

result = await agent.run(...)  # Now shows detailed logs
```

### Correct Mental Model

Agents don't respond without a **working model client**.
- Model client makes API calls
- API calls take time (especially first call)
- Network/auth issues block response
- Always test model client separately first

---

## Issue 6: Tool Execution Failures

### Error Messages
```
ToolCallSummaryMessage: Tool call failed
TypeError: tool_func() takes 1 positional argument but 2 were given
```

### Why This Happens

- Tool signature doesn't match call
- Tool exception not handled
- Missing tool in team
- Wrong parameter names

### Quick Fixes

```python
# WRONG - Wrong signature
def calculate(x):
    return x * 2

agent = AssistantAgent(
    name="math",
    tools=[calculate]
)
# Agent tries to call: calculate(x=5, operation="double")
# ✗ Crashes! Function only takes x

# RIGHT - Accept kwargs
def calculate(x, operation="multiply"):
    if operation == "multiply":
        return x * 2
    return x
```

### Tool Definition Best Practices

```python
from autogen_agentchat.tools import Tool

# WRONG - No docstring (agent doesn't know what it does)
def add(a, b):
    return a + b

# RIGHT - Clear docstring
def add(a: int, b: int) -> int:
    """Add two numbers together.
    
    Args:
        a: First number
        b: Second number
    
    Returns:
        Sum of a and b
    """
    return a + b

tool = Tool(
    name="add",
    description="Add two numbers",
    func=add
)
```

### Debug Tool Calls

```python
result = await agent.run(...)

# Find tool calls
from autogen_agentchat.messages import ToolCallSummaryMessage

for msg in result.messages:
    if isinstance(msg, ToolCallSummaryMessage):
        print(f"Tool: {msg.tool_name}")
        print(f"Args: {msg.tool_call_args}")
        print(f"Result: {msg.tool_call_result}")
        if "Error" in str(msg.tool_call_result):
            print("  ✗ FAILED!")
```

### Correct Mental Model

Tools are **function calls from agents**.
- Function signature must match agent's call
- Docstring tells agent how to use tool
- Exceptions in tools are captured and reported
- Test tools outside agent before deploying

---

## Issue 7: State Loading Errors

### Error Messages
```
json.JSONDecodeError: Expecting value
FileNotFoundError: No such file or directory
```

### Why This Happens

- State file corrupted
- File path wrong
- State format incompatible
- File permissions issue

### Quick Fixes

```python
# WRONG - File doesn't exist
state = agent.load_state("missing_file.json")

# RIGHT - Check first
import os
path = "checkpoint.json"
if os.path.exists(path):
    state = agent.load_state(path)
else:
    print("No checkpoint found, starting fresh")
```

### Save State Properly

```python
import json

# Save
state_dict = {
    "agent_state": agent.state,  # Agent's internal state
    "messages": [msg.model_dump() for msg in result.messages],
    "metadata": {"timestamp": "2024-01-01"}
}

with open("checkpoint.json", "w") as f:
    json.dump(state_dict, f)

# Load
with open("checkpoint.json", "r") as f:
    state_dict = json.load(f)
    
agent.load_state(state_dict)
```

### Verify State File

```python
import json

# Check if file is valid JSON
path = "state.json"
try:
    with open(path, "r") as f:
        data = json.load(f)
    print(f"✓ Valid JSON with keys: {data.keys()}")
except json.JSONDecodeError as e:
    print(f"✗ Invalid JSON: {e}")
    
    # Show first 200 chars
    with open(path, "r") as f:
        content = f.read()
    print(f"Content preview: {content[:200]}")
```

### Correct Mental Model

State files are **JSON serialization of execution history**.
- Must be valid JSON format
- Must exist at specified path
- May need migration if agent code changed
- Always backup before loading

---

## Issue 8: Memory Leaks

### Symptoms

```
Process memory usage: 100MB → 500MB → 2GB
Running for hours: ✗ Crashes
Running same code with limit: ✓ Works
```

### Why This Happens

- Message history grows unbounded
- Agent keeps references to all old messages
- Tools returning huge outputs
- Model client caching responses

### Quick Fixes

```python
# WRONG - Unlimited growth
team = RoundRobinGroupChat(
    participants=[...],
    max_turns=None  # ← Memory grows forever
)

# RIGHT - Bounded turns and messages
team = RoundRobinGroupChat(
    participants=[...],
    max_turns=100,
    termination_condition=MaxMessageTermination(max_messages=500)
)
```

### Monitor Memory

```python
import psutil
import os

process = psutil.Process(os.getpid())

# Check before
mem_before = process.memory_info().rss / 1024 / 1024  # MB

result = await team.run(...)

# Check after
mem_after = process.memory_info().rss / 1024 / 1024
print(f"Memory used: {mem_after - mem_before:.1f} MB")

# If growing too fast:
if (mem_after - mem_before) > 100:  # > 100MB per run
    print("⚠️ WARNING: Possible memory leak!")
```

### Clean Up Messages

```python
# Clear old messages from result
if len(result.messages) > 100:
    # Keep only recent messages
    result.messages = result.messages[-50:]
    
# Save only recent messages to disk
import json
with open("recent_messages.json", "w") as f:
    json.dump(
        [msg.model_dump() for msg in result.messages[-50:]],
        f
    )
```

### Correct Mental Model

Messages accumulate in **RAM during execution**.
- Each message is stored in memory
- Long-running agents can run out of memory
- Bounds on max_turns and max_messages prevent this
- Archive old messages to disk to free RAM

---

## Issue 9: Performance Degradation

### Symptoms

```
First run: 5 seconds
10th run: 30 seconds
100th run: 5 minutes
```

### Why This Happens

- Model client connection pooling issues
- Message history growing (slower to process)
- Model provider throttling
- Resource exhaustion (CPU, network)

### Quick Fixes

```python
# Create fresh client for each run
for i in range(100):
    # WRONG - Reuse same client
    result = await team.run(...)
    
# BETTER - Fresh client each time
for i in range(100):
    client = OpenAIClient(model="gpt-4-turbo")
    agent = AssistantAgent(name="agent", model_client=client)
    result = await agent.run(...)
```

### Measure Performance

```python
import time

times = []
for i in range(5):
    start = time.time()
    result = await team.run(...)
    elapsed = time.time() - start
    times.append(elapsed)
    print(f"Run {i+1}: {elapsed:.1f}s")

# Detect trend
if times[-1] > times[0] * 2:
    print("⚠️ Performance degrading!")
```

### Optimize

```python
# Reduce model calls
# WRONG - Model decides routing every turn
team = SelectorGroupChat(...)  # Calls model for each turn

# BETTER - Fixed routing
team = RoundRobinGroupChat(...)  # No extra model calls

# Reduce message size
# WRONG - Full history
result = await team.run(...)

# BETTER - Summarize old messages
for i, msg in enumerate(result.messages[:-10]):
    if i % 50 == 0:
        result.messages[i] = TextMessage(
            content="[50 messages summarized]"
        )
```

### Correct Mental Model

Performance degrades when **processing grows faster than optimization**.
- More messages = slower processing
- More API calls = more latency
- More runs = potential connection issues
- Monitor and bound growth to prevent degradation

---

## Debugging Checklist

When something breaks, use this checklist:

- [ ] Is model_client initialized?
- [ ] Is API key set and valid?
- [ ] Does model exist in your plan?
- [ ] Is conversation within token limit?
- [ ] Do agents have system messages?
- [ ] Do tools have docstrings?
- [ ] Is termination_condition set?
- [ ] Are max_turns bounded?
- [ ] Does result have messages?
- [ ] Check `result.stop_reason` — why did it stop?

---

## Getting Help

### Enable Debug Logging
```python
import logging
logging.basicConfig(level=logging.DEBUG)

# Now see detailed logs
result = await team.run(...)
```

### Save Complete Context
```python
import json

debug_context = {
    "error": str(exception),
    "messages": [msg.model_dump() for msg in result.messages],
    "stop_reason": str(result.stop_reason),
    "model": agent.model_client.model,
    "participants": [a.name for a in team.participants]
}

with open("debug_context.json", "w") as f:
    json.dump(debug_context, f, indent=2)
```

### Report Issues
Include:
- Error message (full traceback)
- Minimal reproducible code
- AutoGen version
- Model and provider used

---

## References

- See [10-single-agent-loop.md](10-single-agent-loop.md) for execution flow
- See [13-selector-group-chat.md](13-selector-group-chat.md) for loop prevention
- See [14-swarm.md](14-swarm.md) for handoff debugging
- Official AutoGen: https://microsoft.github.io/autogen/
