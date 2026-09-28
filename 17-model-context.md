# Part 17: Model Context Management

## What It Is

**Context** is the information you send to the LLM. Managing it well = cheaper, faster, better responses.

```
Conversation history:
Turn 1: [Message] (200 tokens)
Turn 2: [Message] (200 tokens)
Turn 3: [Message] (200 tokens)
...
Turn 100: [Message] (200 tokens)

Total context: 20,000 tokens
↓ Problem: Expensive and slow!
↓ Solution: Trim old messages, summarize, or compress
```

---

## Context Window Limits

Each model has a maximum:

| Model | Limit | Cost per 1K input | Cost per 1K output |
|-------|-------|------|------|
| GPT-4 | 8K | $0.03 | $0.06 |
| GPT-4 Turbo | 128K | $0.01 | $0.03 |
| Claude 3.5 Sonnet | 200K | $0.003 | $0.015 |

**Problem**: Long conversations waste tokens and money

---

## Token Counting

### Estimate Tokens

```python
# Rough rule: 1 token ≈ 4 characters
message = "This is a test message"
estimated_tokens = len(message) // 4  # ~5 tokens

# For accuracy, use model's tokenizer
from autogen_agentchat.model_client import count_tokens

tokens = count_tokens("Your message here", model="gpt-4")
```

### Check Total Context

```python
def calculate_context_tokens(messages):
    """Estimate tokens in conversation"""
    total = 0
    for msg in messages:
        # Each message has some overhead
        msg_tokens = count_tokens(msg.content) + 10  # 10 for metadata
        total += msg_tokens
    return total

result = await team.run(...)
context_used = calculate_context_tokens(result.messages)
print(f"Context used: {context_used} tokens")

# If near limit, truncate next conversation
if context_used > 7000:  # Out of 8K
    print("WARNING: Near context limit!")
```

---

## Context Pruning

### Strategy 1: Keep Recent Messages Only

```python
def keep_recent_messages(messages, keep_count=10):
    """Keep only last N messages"""
    if len(messages) > keep_count:
        # Keep recent only
        trimmed = messages[-keep_count:]
        # Could add summary of old messages
        return trimmed
    return messages

# Usage
result = await team.run(...)
if len(result.messages) > 50:
    trimmed = keep_recent_messages(result.messages, keep_count=20)
```

### Strategy 2: Summary of Old Messages

```python
from autogen_agentchat.messages import TextMessage

def summarize_old_messages(messages, keep_count=10, summary_model=None):
    """Summarize old messages to preserve context"""
    
    if len(messages) <= keep_count:
        return messages
    
    old_messages = messages[:-keep_count]
    recent_messages = messages[-keep_count:]
    
    # Create summary message
    summary_prompt = "Summarize this conversation:\n"
    for msg in old_messages:
        summary_prompt += f"{msg.source}: {msg.content}\n"
    
    # Get summary from model (if available)
    # For now, just create a placeholder
    summary_msg = TextMessage(
        content=f"[Summary of {len(old_messages)} earlier messages]",
        source="system"
    )
    
    return [summary_msg] + recent_messages

# Usage
trimmed = summarize_old_messages(result.messages)
```

### Strategy 3: Selective Truncation

```python
def smart_truncate(messages, target_tokens=6000):
    """Truncate to fit token budget"""
    
    if calculate_context_tokens(messages) <= target_tokens:
        return messages
    
    # Start from oldest, remove until under budget
    for i in range(len(messages)):
        trimmed = messages[i:]
        if calculate_context_tokens(trimmed) <= target_tokens:
            return trimmed
    
    # Last resort: keep only last message
    return messages[-1:]
```

---

## Context Strategies

### Strategy: System Message at Top

```python
# Always include critical system context
full_context = [
    TextMessage(
        content="SYSTEM: You are a code reviewer. Focus on: 1) Correctness, 2) Performance, 3) Readability",
        source="system"
    ),
    # ... then conversation messages
]
```

### Strategy: Few-Shot Examples

```python
# Include examples for better performance
context_with_examples = [
    TextMessage(content="EXAMPLE 1:", source="system"),
    TextMessage(content="Code: def add(a, b): return a+b", source="user"),
    TextMessage(content="Review: Good. Simple and correct.", source="reviewer"),
    
    TextMessage(content="EXAMPLE 2:", source="system"),
    TextMessage(content="Code: result = x / 0", source="user"),
    TextMessage(content="Review: ERROR: Division by zero risk", source="reviewer"),
    
    # ... then actual conversation
]
```

---

## Compressed Representations

### Idea: Store Messages Compressed

```python
import gzip
import json

# Compress old messages
def compress_messages(messages):
    """Store messages compressed"""
    data = [m.model_dump() for m in messages]
    json_str = json.dumps(data)
    compressed = gzip.compress(json_str.encode())
    return compressed

# Decompress when needed
def decompress_messages(compressed):
    """Restore from compressed"""
    json_str = gzip.decompress(compressed).decode()
    data = json.loads(json_str)
    # Reconstruct message objects
    return data
```

---

## Trade-offs: Quality vs Cost

| Approach | Cost | Quality | Speed |
|----------|------|---------|-------|
| **Full history** | High | Best | Slow |
| **Recent only** | Low | Medium | Fast |
| **Summarized** | Medium | Good | Medium |
| **Examples** | Medium | Good | Slower |

**Decision**: What's your priority?
- **Accuracy critical**: Use full or summarized
- **Cost critical**: Use recent only
- **Balanced**: Use summarized

---

## Implementation Pattern

```python
async def run_with_context_management(team, task, max_tokens=7000):
    """Run team while managing context"""
    
    all_messages = []
    
    async for event in team.run_stream(task=task):
        if event["type"] == "message_created":
            all_messages.append(event["message"])
            
            # Check context size
            context_tokens = calculate_context_tokens(all_messages)
            
            if context_tokens > max_tokens:
                print(f"Context growing ({context_tokens} tokens), trimming...")
                all_messages = keep_recent_messages(all_messages, keep_count=15)
        
        elif event["type"] == "team_finished":
            return event["result"]
```

---

## Custom Context Handler

```python
from autogen_agentchat.agents import BaseChatAgent

class ContextAwareAgent(BaseChatAgent):
    """Agent that manages its own context"""
    
    def __init__(self, name, model_client, context_limit=4000):
        self._name = name
        self.model_client = model_client
        self.context_limit = context_limit
    
    @property
    def name(self):
        return self._name
    
    async def on_messages(self, messages, sender):
        """Process with context management"""
        
        # Trim if needed
        if calculate_context_tokens(messages) > self.context_limit:
            messages = keep_recent_messages(messages, keep_count=10)
        
        # Process with trimmed context
        response = TextMessage(
            content=f"Processed {len(messages)} messages",
            source=self.name
        )
        return [response], "ok"
```

---

## Monitoring Context

```python
def monitor_context(result):
    """Report on context usage"""
    
    tokens_per_turn = []
    cumulative_tokens = 0
    
    for msg in result.messages:
        tokens = count_tokens(msg.content) + 10
        cumulative_tokens += tokens
        tokens_per_turn.append(cumulative_tokens)
    
    # Report
    print(f"Messages: {len(result.messages)}")
    print(f"Total tokens: {cumulative_tokens}")
    print(f"Avg per message: {cumulative_tokens // len(result.messages)}")
    print(f"Peak tokens: {max(tokens_per_turn) if tokens_per_turn else 0}")
    
    # Warn if approaching limit
    if cumulative_tokens > 6000:
        print("WARNING: Approaching context limit!")
```

---

## When Context Management Matters

**Matters**:
- Long conversations (50+ messages)
- Expensive models (GPT-4)
- Production systems (cost control)
- Repeated runs (each adds context)

**Doesn't matter**:
- Short conversations (< 10 messages)
- Cheap models (GPT-3.5)
- One-off research
- Small context windows

---

## Common Mistakes

### Mistake 1: Not Tracking Context Size
```python
# WRONG - No monitoring
for i in range(100):
    result = await agent.run(task=f"Task {i}")
    # Context grows silently!

# RIGHT - Monitor
for i in range(100):
    result = await agent.run(task=f"Task {i}")
    tokens = calculate_context_tokens(result.messages)
    if tokens > 5000:
        print(f"Context high: {tokens} tokens")
```

### Mistake 2: Aggressive Trimming
```python
# WRONG - Loses important context
messages = messages[-1:]  # Keep only last message!

# RIGHT - Balance
messages = messages[-20:]  # Keep recent 20
```

### Mistake 3: Trimming at Wrong Time
```python
# WRONG - Trim in middle of conversation
# This confuses the agent (context disappears mid-task)

# RIGHT - Trim between runs
result1 = await team.run(task="A")
messages = keep_recent_messages(result1.messages)
# Use trimmed for next run
```

---

## Next Steps

- [18-memory.md](18-memory.md) - Long-term memory
- [19-state-persistence.md](19-state-persistence.md) - Saving context
- [22-performance-cost.md](22-performance-cost.md) - Cost optimization

---

## References

- See [31-runtime-outputs-objects.md](31-runtime-outputs-objects.md) for message structure
- Official AutoGen: https://microsoft.github.io/autogen/
