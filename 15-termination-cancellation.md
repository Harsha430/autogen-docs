# Part 15: Termination & Cancellation

## What It Is

**Termination** stops the team when work is done. **Cancellation** stops it immediately if needed.

```
Execution running...
  ↓ (condition met)
Team terminates gracefully
  ↓
return TaskResult
```

---

## When to Use

✅ Prevent infinite loops (max message count)  
✅ Stop when goal achieved (specific word)  
✅ Emergency stop (user Ctrl+C)  
✅ Timeout handling  

---

## Termination vs Cancellation

| Aspect | Termination | Cancellation |
|--------|------------|------------|
| **Trigger** | Condition met | External signal |
| **Timing** | Graceful (finish current turn) | Immediate |
| **Use** | Work complete | Emergency stop |
| **Example** | Message says "DONE" | Timeout, user interrupt |

---

## TerminationCondition Types

### MaxMessageTermination

```python
from autogen_agentchat.termination import MaxMessageTermination

# Stop after N messages
termination = MaxMessageTermination(max_messages=50)

team = RoundRobinGroupChat(
    participants=[...],
    termination_condition=termination
)

result = await team.run(task="...")
# Stops after 50 total messages
```

**Usage**: Safety limit (prevent runaway)

### StopMessageTermination

```python
from autogen_agentchat.termination import StopMessageTermination

# Stop when message contains keyword
termination = StopMessageTermination(
    stop_words=["DONE", "COMPLETE", "APPROVED"]
)

team = RoundRobinGroupChat(
    participants=[...],
    termination_condition=termination
)

result = await team.run(task="...")
# Stops when any agent says "DONE"
```

**Usage**: Goal-based termination

### TextMentionTermination

```python
# Stop if specific text mentioned
termination = StopMessageTermination(
    stop_words=["ERROR", "FAILED", "ABORT"]
)

# Stops if any error mentioned
```

---

## Custom Termination Conditions

### Simple Custom

```python
from autogen_agentchat.termination import TerminationCondition

class CustomTermination(TerminationCondition):
    """Stop after N turns"""
    
    def __init__(self, max_turns: int = 10):
        self.max_turns = max_turns
    
    async def __call__(self, messages: list) -> bool:
        """Return True to terminate"""
        # Count turns (rough: each agent speaks = 1 turn)
        return len(messages) >= self.max_turns

# Usage
termination = CustomTermination(max_turns=20)
team = RoundRobinGroupChat(
    participants=[...],
    termination_condition=termination
)
```

### Advanced Custom: Message Pattern

```python
import re
from autogen_agentchat.termination import TerminationCondition

class PatternTermination(TerminationCondition):
    """Stop if message matches regex"""
    
    def __init__(self, pattern: str):
        self.pattern = re.compile(pattern, re.IGNORECASE)
    
    async def __call__(self, messages: list) -> bool:
        if not messages:
            return False
        
        latest = messages[-1].content
        return bool(self.pattern.search(latest))

# Usage
termination = PatternTermination(r"decision.*made|approved|rejected")
```

### Complex Custom: Convergence Detection

```python
from autogen_agentchat.termination import TerminationCondition
from collections import Counter

class ConvergenceTermination(TerminationCondition):
    """Stop when agents agree"""
    
    def __init__(self, agreement_threshold: float = 0.8, window_size: int = 5):
        self.threshold = agreement_threshold
        self.window = window_size
    
    async def __call__(self, messages: list) -> bool:
        if len(messages) < self.window:
            return False
        
        # Get last N messages
        recent = messages[-self.window:]
        
        # Count unique responses
        responses = [m.content[:20] for m in recent]  # First 20 chars
        counter = Counter(responses)
        
        # Get most common
        most_common_count = counter.most_common(1)[0][1]
        
        # If > threshold all say same thing, converged
        convergence = most_common_count / len(recent)
        return convergence > self.threshold
```

---

## Combining Conditions

### MaxMessage OR StopWord

```python
from autogen_agentchat.termination import MaxMessageTermination, StopMessageTermination

team = RoundRobinGroupChat(
    participants=[...],
    max_turns=20,  # Safety limit
    termination_condition=StopMessageTermination(
        stop_words=["COMPLETE"]
    )
)

# Stops when EITHER:
# - max_turns (20) reached, OR
# - Message contains "COMPLETE"
```

### Multiple Conditions (Custom)

```python
class MultiCondition:
    """Combine multiple termination conditions"""
    
    def __init__(self, *conditions):
        self.conditions = conditions
    
    async def __call__(self, messages):
        # Stop if ANY condition is true
        for condition in self.conditions:
            if await condition(messages):
                return True
        return False

# Usage
termination = MultiCondition(
    MaxMessageTermination(max_messages=100),
    StopMessageTermination(stop_words=["DONE"]),
    ConvergenceTermination(threshold=0.9)
)
```

---

## Cancellation: Emergency Stop

### Timeout-Based Cancellation

```python
import asyncio

async def run_with_timeout():
    """Stop if takes too long"""
    
    try:
        result = await asyncio.wait_for(
            team.run(task="..."),
            timeout=300  # 5 minutes
        )
        return result
    
    except asyncio.TimeoutError:
        print("Execution timed out!")
        return None
```

### User Interrupt

```python
import asyncio
import signal

def handle_interrupt(signum, frame):
    """Handle Ctrl+C"""
    print("Interrupted by user")
    raise KeyboardInterrupt

signal.signal(signal.SIGINT, handle_interrupt)

try:
    result = await team.run(task="...")
except KeyboardInterrupt:
    print("Execution cancelled")
```

### Cancellation Token

```python
import asyncio

async def run_with_cancellation():
    """Manual cancellation"""
    
    task = asyncio.create_task(team.run(task="..."))
    
    # After 30 seconds, cancel
    await asyncio.sleep(30)
    task.cancel()
    
    try:
        result = await task
    except asyncio.CancelledError:
        print("Task cancelled")
        return None
```

---

## Examples

### Example 1: Research with Max Messages

```python
from autogen_agentchat.termination import MaxMessageTermination

termination = MaxMessageTermination(max_messages=100)

team = Swarm(
    participants=[researcher, analyzer, writer],
    termination_condition=termination,
    max_turns=30
)

result = await team.run(
    task="Research and write article on quantum computing"
)

print(f"Stopped: {result.stop_reason}")
print(f"Messages: {len(result.messages)}")
```

### Example 2: Approval Loop with Stop Word

```python
from autogen_agentchat.termination import StopMessageTermination

termination = StopMessageTermination(
    stop_words=["APPROVED", "REJECTED"]
)

team = RoundRobinGroupChat(
    participants=[creator, reviewer],
    max_turns=10,  # Safety: max 10 turns
    termination_condition=termination
)

result = await team.run(
    task="Create and review proposal"
)

# Stops when reviewer says "APPROVED" or "REJECTED"
final_msg = result.messages[-1]
print(f"Decision: {final_msg.content}")
```

### Example 3: Timeout on Expensive Operation

```python
async def expensive_analysis():
    """Stop if takes too long"""
    
    try:
        result = await asyncio.wait_for(
            team.run(
                task="Analyze 10GB dataset"
            ),
            timeout=600  # 10 minutes max
        )
        return result
    
    except asyncio.TimeoutError:
        print("Analysis took too long. Stopping.")
        return None
```

---

## Error Handling During Termination

### Detect Why It Stopped

```python
result = await team.run(...)

if result.stop_reason == "termination_condition":
    print("Goal achieved!")

elif result.stop_reason == "max_turns":
    print("Hit max turns limit")

elif result.stop_reason == "error":
    print(f"Error occurred: {result.error}")

else:
    print(f"Stopped: {result.stop_reason}")
```

### Graceful Degradation

```python
result = await team.run(...)

if result.stop_reason == "max_messages":
    # Got incomplete results
    print(f"Got {len(result.messages)} messages (incomplete)")
    # Use what we have
    final = result.messages[-1].content
else:
    # Complete results
    final = result.messages[-1].content

print(f"Result: {final}")
```

---

## Common Mistakes

### Mistake 1: No Termination (Infinite Loop)
```python
# WRONG - Will run forever
team = RoundRobinGroupChat(
    participants=[A, B],
    max_turns=None,  # No limit!
    termination_condition=None  # No condition!
)

# RIGHT - Always have a limit
team = RoundRobinGroupChat(
    participants=[A, B],
    max_turns=20,
    termination_condition=MaxMessageTermination(max_messages=50)
)
```

### Mistake 2: Stop Word Never Matches
```python
# WRONG - Agent never says "DONE"
termination = StopMessageTermination(stop_words=["DONE"])

# Agent says: "Finished"
# ✗ Doesn't match, runs forever

# RIGHT - Stop words that agent will say
termination = StopMessageTermination(
    stop_words=["Finished", "Complete", "Done"]
)

# And tell agent to say them:
agent_system = """
When complete, say: "Finished processing"
"""
```

### Mistake 3: Timeout Too Aggressive
```python
# WRONG - 1 second timeout (too short)
result = await asyncio.wait_for(
    team.run(...),
    timeout=1  # Almost always times out
)

# RIGHT - Realistic timeout
result = await asyncio.wait_for(
    team.run(...),
    timeout=300  # 5 minutes reasonable
)
```

### Mistake 4: Ignoring Cancellation

```python
# WRONG - Cancellation not handled
try:
    result = await team.run(...)
except KeyboardInterrupt:
    pass  # Silently ignore

# RIGHT - Clean up or notify
try:
    result = await team.run(...)
except KeyboardInterrupt:
    print("User cancelled execution")
    # Could save partial results, notify user, etc.
```

---

## Termination Debugging

```python
# Check why it stopped
result = await team.run(...)

print(f"Stop reason: {result.stop_reason}")
print(f"Message count: {len(result.messages)}")
print(f"Last message: {result.messages[-1].content[:100]}")

# For custom conditions, add logging
class DebugTermination(TerminationCondition):
    async def __call__(self, messages):
        should_stop = len(messages) > 20
        if should_stop:
            print(f"Stopping: {len(messages)} messages > 20")
        return should_stop
```

---

## Next Steps

- [16-state.md](16-state.md) - State management
- [28-troubleshooting.md](28-troubleshooting.md) - Infinite loop debugging
- [20-human-in-loop.md](20-human-in-loop.md) - User interruption

---

## References

- See [11-teams.md](11-teams.md) for team basics
- Official AutoGen: https://microsoft.github.io/autogen/
