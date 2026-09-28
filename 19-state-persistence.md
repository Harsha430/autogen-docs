# Part 19: State Persistence - Save and Resume

## What It Is

**State persistence** lets you save agent/team state and resume later, recovering full context.

```
Conversation stopped at Turn 50
  ↓ (save state)
Days pass...
  ↓ (load state)
Resume at Turn 51
```

---

## Why Persist State

✅ Resume long workflows  
✅ Recover from crashes  
✅ Parallel workflows (each saves state)  
✅ Audit trail (who did what)  
✅ Distributed systems  

---

## Serialization Formats

### JSON Format

```python
import json
from autogen_agentchat.messages import TextMessage

def serialize_to_json(result):
    """Convert TaskResult to JSON"""
    
    state_dict = {
        "metadata": {
            "stop_reason": result.stop_reason,
            "message_count": len(result.messages),
            "timestamp": datetime.now().isoformat()
        },
        "messages": [
            {
                "type": type(msg).__name__,
                "source": msg.source,
                "content": msg.content if hasattr(msg, "content") else str(msg)
            }
            for msg in result.messages
        ]
    }
    
    return json.dumps(state_dict, indent=2)

# Save
result = await team.run(task="...")
json_state = serialize_to_json(result)

with open("checkpoint.json", "w") as f:
    f.write(json_state)
```

### File Structure

```json
{
  "metadata": {
    "stop_reason": "max_turns",
    "message_count": 45,
    "timestamp": "2024-01-01T10:00:00",
    "version": 1
  },
  "messages": [
    {
      "type": "TextMessage",
      "source": "agent_a",
      "content": "Hello, let me analyze this..."
    },
    {
      "type": "HandoffMessage",
      "source": "agent_a",
      "to": "agent_b",
      "content": "Please take it from here"
    }
  ]
}
```

---

## Save and Resume Pattern

### Save State

```python
import json
from datetime import datetime

async def save_checkpoint(result, filename):
    """Save execution state"""
    
    checkpoint = {
        "timestamp": datetime.now().isoformat(),
        "messages": [msg.model_dump() for msg in result.messages],
        "stop_reason": result.stop_reason,
        "metadata": {
            "total_turns": len(result.messages),
            "last_speaker": result.messages[-1].source if result.messages else None
        }
    }
    
    with open(filename, "w") as f:
        json.dump(checkpoint, f, indent=2)
    
    print(f"Saved checkpoint: {filename}")

# Usage
result = await team.run(task="Process report")
await save_checkpoint(result, "report_checkpoint.json")
```

### Load State

```python
def load_checkpoint(filename):
    """Load execution state"""
    
    with open(filename, "r") as f:
        checkpoint = json.load(f)
    
    # Reconstruct messages
    messages = []
    for msg_data in checkpoint["messages"]:
        # Simplified: just extract content
        messages.append({
            "source": msg_data.get("source"),
            "content": msg_data.get("content"),
            "type": msg_data.get("type")
        })
    
    return checkpoint, messages

# Usage
checkpoint, messages = load_checkpoint("report_checkpoint.json")
print(f"Loaded {len(messages)} messages")
print(f"Last stopped: {checkpoint['stop_reason']}")
```

### Resume from Checkpoint

```python
async def resume_from_checkpoint(team, checkpoint_file):
    """Resume interrupted execution"""
    
    checkpoint, messages = load_checkpoint(checkpoint_file)
    
    if checkpoint["stop_reason"] == "max_turns":
        print("Previous run hit max turns, resuming...")
    elif checkpoint["stop_reason"] == "terminated":
        print("Work complete, not resuming")
        return messages
    
    # Continue with fresh run (now with more context from checkpoint)
    result = await team.run(task="Continue processing")
    
    # Append to checkpoint
    checkpoint["messages"].extend(result.messages)
    checkpoint["timestamp"] = datetime.now().isoformat()
    
    return checkpoint["messages"]
```

---

## State Versioning

### Problem: Code Changes

```
Version 1 of code:
  State format: {"turn": 5, "messages": [...]}

Version 2 of code:
  State format: {"turn": 5, "messages": [...], "metadata": {...}}

Old state incompatible!
```

### Solution: Migration

```python
CURRENT_STATE_VERSION = 2

def migrate_state(old_state):
    """Upgrade old state to new format"""
    
    if old_state.get("version") == 1:
        # Upgrade from v1 to v2
        old_state["version"] = 2
        old_state["metadata"] = {
            "migrated_from": "v1",
            "migrated_at": datetime.now().isoformat()
        }
    
    if old_state.get("version") == 2:
        return old_state  # Already current
    
    raise ValueError(f"Unknown state version: {old_state.get('version')}")

# Usage
checkpoint = load_checkpoint("old_checkpoint.json")
checkpoint = migrate_state(checkpoint)
# Now safe to use with new code
```

---

## Resuming Interrupted Execution

### Scenario: Timeout

```python
async def run_with_resume():
    """Run with timeout, save if interrupted"""
    
    try:
        result = await asyncio.wait_for(
            team.run(task="Process large dataset"),
            timeout=600  # 10 minutes
        )
        return result
    
    except asyncio.TimeoutError:
        print("Timeout! Saving state...")
        # Save partial result
        await save_checkpoint(result, "timeout_checkpoint.json")
        return None

# Later, resume
if os.path.exists("timeout_checkpoint.json"):
    print("Found interrupted work, resuming...")
    checkpoint, messages = load_checkpoint("timeout_checkpoint.json")
    # Continue work
```

### Scenario: User Interrupt (Ctrl+C)

```python
import signal

result = None

def save_on_interrupt(signum, frame):
    """Save state on Ctrl+C"""
    global result
    print("Interrupted! Saving state...")
    if result:
        asyncio.run(save_checkpoint(result, "interrupt_checkpoint.json"))
    raise KeyboardInterrupt

signal.signal(signal.SIGINT, save_on_interrupt)

try:
    result = await team.run(task="...")
except KeyboardInterrupt:
    print("Saved. Resume later with: await resume_from_checkpoint(...)")
```

---

## Distributed State Management

### Multi-Machine Setup

```
Machine A: Agent runs, saves state to shared storage
  ↓
Machine B: Reads state, continues execution
  ↓
Machine C: Reads state, runs in parallel
```

### Cloud Storage

```python
import boto3  # AWS S3 example

def save_state_to_s3(state_dict, bucket, key):
    """Save to cloud"""
    s3 = boto3.client('s3')
    s3.put_object(
        Bucket=bucket,
        Key=key,
        Body=json.dumps(state_dict)
    )
    print(f"Saved to s3://{bucket}/{key}")

def load_state_from_s3(bucket, key):
    """Load from cloud"""
    s3 = boto3.client('s3')
    response = s3.get_object(Bucket=bucket, Key=key)
    return json.loads(response['Body'].read())

# Usage
await save_checkpoint(result, "checkpoint.json")
state = load_state_from_s3("my-bucket", "checkpoints/checkpoint.json")
```

---

## What to Persist vs Compute

### Persist (Large, Computed Once)
```
- Message history (expensive to regenerate)
- Tool results (may be stale if recomputed)
- User decisions (immutable)
- Metadata (timestamps, decisions)
```

### Don't Persist (Cheap, Auto-Recoverable)
```
- Derived statistics (recompute from messages)
- Model embeddings (can regenerate)
- Cache entries (refresh on load)
- Log entries (non-critical)
```

```python
def smart_save(result):
    """Save only essential state"""
    
    # SAVE
    essential = {
        "messages": result.messages,  # Can't regenerate
        "stop_reason": result.stop_reason,  # Important for resume logic
        "timestamp": datetime.now().isoformat()
    }
    
    # DON'T SAVE (can regenerate)
    # "statistics": compute from messages
    # "embeddings": recompute if needed
    
    return essential
```

---

## Common Mistakes

### Mistake 1: Partial Save

```python
# WRONG - Only save some messages
for msg in result.messages[:10]:  # Only first 10!
    checkpoint.append(msg)

# RIGHT - Save all
checkpoint = [msg for msg in result.messages]
```

### Mistake 2: Not Versioning State

```python
# WRONG - No version tracking
checkpoint = {
    "messages": [...],
    "metadata": {...}
}

# RIGHT - Include version
checkpoint = {
    "version": 2,
    "messages": [...],
    "metadata": {...}
}
```

### Mistake 3: Forgetting to Deserialize

```python
# WRONG - Using JSON as-is
checkpoint = json.load(f)
msg_obj = checkpoint["messages"][0]  # This is dict, not message object!

# RIGHT - Reconstruct
msg_dict = checkpoint["messages"][0]
msg_obj = TextMessage(
    content=msg_dict["content"],
    source=msg_dict["source"]
)
```

### Mistake 4: Stale State in Concurrent Setup

```python
# WRONG - Multiple processes save simultaneously
task1 = save_checkpoint(result1, "checkpoint.json")  # Both save to
task2 = save_checkpoint(result2, "checkpoint.json")  # Same file!
await asyncio.gather(task1, task2)  # Race condition

# RIGHT - Unique filenames
await asyncio.gather(
    save_checkpoint(result1, f"checkpoint_{uuid.uuid4()}.json"),
    save_checkpoint(result2, f"checkpoint_{uuid.uuid4()}.json")
)
```

---

## Recovery Patterns

### Cold Start (No State)
```python
async def start_fresh():
    result = await team.run(task="Start new work")
    await save_checkpoint(result, "checkpoint.json")
    return result
```

### Resume (Have State)
```python
async def start_or_resume():
    if os.path.exists("checkpoint.json"):
        return await resume_from_checkpoint(team, "checkpoint.json")
    else:
        return await start_fresh()
```

### Cleanup Old Checkpoints
```python
import os
import time

def cleanup_old_checkpoints(pattern="checkpoint_*.json", days=7):
    """Remove checkpoints older than N days"""
    cutoff = time.time() - (days * 86400)
    
    for filepath in glob.glob(pattern):
        if os.path.getmtime(filepath) < cutoff:
            os.remove(filepath)
            print(f"Deleted: {filepath}")
```

---

## Next Steps

- [16-state.md](16-state.md) - State management basics
- [17-model-context.md](17-model-context.md) - Context in state
- [25-complete-project.md](25-complete-project.md) - Full example with persistence

---

## References

- Official AutoGen: https://microsoft.github.io/autogen/
