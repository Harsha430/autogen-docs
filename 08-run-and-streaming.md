# Part 8: Running Agents and Streaming

## What It Is

AutoGen offers two ways to execute agents:
- **run()**: Wait for complete result
- **run_stream()**: Get events in real-time

```python
# Run: Get full result
result = await agent.run(task="...")
print(result.messages)

# Stream: Get events as they happen
async for event in agent.run_stream(task="..."):
    print(f"Event: {event}")
```

---

## run(): Complete Execution

### Basic Usage

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat

team = RoundRobinGroupChat(
    participants=[agent_a, agent_b],
    max_turns=10
)

result = await team.run(task="Analyze the dataset")
print(result.stop_reason)  # "max_turns" or termination condition
print(result.messages)     # All messages generated
```

### Return Value: TaskResult

```python
from autogen_agentchat.runtime import TaskResult

result: TaskResult = await team.run(task="...")

# Properties
print(result.messages)      # List of all messages
print(result.stop_reason)   # Why execution stopped
```

### Stop Reasons

| Stop Reason | Meaning |
|------------|---------|
| `"message_processed"` | Agent processed message |
| `"max_turns"` | Reached max_turns limit |
| `"termination_condition"` | Termination condition triggered |
| `"error"` | Error occurred |
| `"tool_call_failed"` | Tool execution failed |

### Example: Run Until Completion

```python
from autogen_agentchat.termination import StopMessageTermination

team = RoundRobinGroupChat(
    participants=[writer, reviewer],
    termination_condition=StopMessageTermination(
        stop_words=["APPROVED", "COMPLETE"]
    )
)

result = await team.run(task="Write report")

# Execution stops when any message contains "APPROVED" or "COMPLETE"
print(f"Stopped: {result.stop_reason}")
print(f"Final message: {result.messages[-1].content}")
```

---

## run_stream(): Real-Time Streaming

### Basic Usage

```python
# Get events as they happen
async for event in team.run_stream(task="Analyze data"):
    print(event)
    # event is a dictionary with event details
```

### Event Types

Common streaming events:

| Event Type | When | Example |
|-----------|------|---------|
| `"agent_starting"` | Agent about to speak | `{"type": "agent_starting", "agent": "analyzer"}` |
| `"message_created"` | Agent created message | `{"type": "message_created", "message": {...}}` |
| `"tool_call_starting"` | Tool about to execute | `{"type": "tool_call_starting", "tool": "search"}` |
| `"tool_call_completed"` | Tool finished | `{"type": "tool_call_completed", "result": "..."}` |
| `"team_finished"` | Execution complete | `{"type": "team_finished", "result": {...}}` |

### Example: Monitor Progress

```python
import asyncio

task = "Write comprehensive report"
print(f"Starting: {task}")

async for event in team.run_stream(task=task):
    event_type = event.get("type")
    
    if event_type == "agent_starting":
        print(f"  → {event['agent']} is speaking...")
    
    elif event_type == "message_created":
        msg = event["message"]
        print(f"    {msg.source}: {msg.content[:60]}...")
    
    elif event_type == "tool_call_starting":
        print(f"    Calling tool: {event['tool']}")
    
    elif event_type == "tool_call_completed":
        print(f"    Tool result: {event['result'][:100]}...")
    
    elif event_type == "team_finished":
        print(f"Done! Reason: {event['result'].stop_reason}")
```

### Collect Results from Stream

```python
# Collect all messages while streaming
messages = []
stop_reason = None

async for event in team.run_stream(task="..."):
    if event.get("type") == "message_created":
        messages.append(event["message"])
    
    elif event.get("type") == "team_finished":
        stop_reason = event["result"].stop_reason

print(f"Collected {len(messages)} messages")
print(f"Stopped: {stop_reason}")
```

---

## Async Patterns

### Pattern 1: Sequential Execution

```python
import asyncio

async def process_sequentially():
    """Run agents one after another"""
    
    result1 = await agent_a.run(task="Task A")
    print(f"A finished: {result1.stop_reason}")
    
    result2 = await agent_b.run(task="Task B")
    print(f"B finished: {result2.stop_reason}")
    
    return result1, result2

# Execute
results = await process_sequentially()
```

### Pattern 2: Parallel Execution

```python
async def process_parallel():
    """Run agents in parallel"""
    
    result_a = asyncio.create_task(agent_a.run(task="Task A"))
    result_b = asyncio.create_task(agent_b.run(task="Task B"))
    
    # Wait for both
    results = await asyncio.gather(result_a, result_b)
    return results

# Execute
results = await process_parallel()
```

### Pattern 3: Timeout

```python
async def run_with_timeout():
    """Run with timeout"""
    
    try:
        result = await asyncio.wait_for(
            team.run(task="..."),
            timeout=60  # 60 seconds
        )
        return result
    
    except asyncio.TimeoutError:
        print("Execution timed out!")
        return None
```

---

## Error Handling During Execution

### Handle Tool Failures

```python
async def run_with_tool_handling():
    """Handle tool failures gracefully"""
    
    async for event in team.run_stream(task="..."):
        if event.get("type") == "tool_call_failed":
            error = event.get("error")
            print(f"Tool failed: {error}")
            # Could retry, skip, or escalate
        
        elif event.get("type") == "team_finished":
            result = event["result"]
            if result.stop_reason == "error":
                print(f"Execution error: {result.error}")
```

### Handle Agent Errors

```python
async def run_with_error_handling():
    """Catch and handle errors"""
    
    try:
        result = await team.run(task="...")
        
        if result.stop_reason == "error":
            print(f"Team error: {result.error}")
            # Handle error
        else:
            print(f"Success: {result.stop_reason}")
    
    except Exception as e:
        print(f"Exception: {type(e).__name__}: {e}")
```

---

## Streaming for Observability

### Real-time Monitoring

```python
import time

async def monitored_run(team, task):
    """Run with performance monitoring"""
    
    start_time = time.time()
    event_count = 0
    
    async for event in team.run_stream(task=task):
        event_count += 1
        elapsed = time.time() - start_time
        
        if event_count % 5 == 0:
            print(f"[{elapsed:.1f}s] Processed {event_count} events")
        
        if event.get("type") == "team_finished":
            total_time = time.time() - start_time
            print(f"Completed in {total_time:.1f}s ({event_count} events)")
            return event["result"]
```

### Live Dashboard (Pseudo-code)

```python
async def stream_to_dashboard(team, task):
    """Stream events to web dashboard"""
    
    async for event in team.run_stream(task=task):
        # Send to websocket, websocket broadcast to browser
        await websocket.send_json({
            "type": event.get("type"),
            "timestamp": time.time(),
            "agent": event.get("agent"),
            "message": str(event.get("message", ""))[:100]
        })
        
        if event.get("type") == "team_finished":
            await websocket.send_json({
                "type": "complete",
                "stop_reason": event["result"].stop_reason
            })
            break
```

---

## run() vs run_stream() Comparison

| Aspect | run() | run_stream() |
|--------|-------|------------|
| **Returns** | Complete TaskResult | Stream of events |
| **Timing** | Waits until done | Real-time events |
| **Use** | Final result needed | Monitoring, debugging |
| **Overhead** | Lower | Higher (more events) |
| **Interruption** | Can't interrupt | Can break early |
| **Example** | Batch processing | Live dashboard |

---

## When to Use Each

### Use run()
```python
# Simple batch processing
result = await agent.run(task="analyze file.csv")
print(result.messages[-1])  # Get final answer
```

### Use run_stream()
```python
# Need real-time updates
async for event in team.run_stream(task="..."):
    if event["type"] == "message_created":
        # Update UI
        ui.display_message(event["message"])
```

---

## Common Patterns

### Pattern: Collect All Events

```python
async def collect_all_events(team, task):
    """Collect all events in a list"""
    
    events = []
    async for event in team.run_stream(task=task):
        events.append(event)
    
    # Now analyze all events
    message_count = sum(1 for e in events if e["type"] == "message_created")
    tool_calls = sum(1 for e in events if e["type"] == "tool_call_starting")
    
    return events, message_count, tool_calls
```

### Pattern: Early Exit

```python
async def run_until_approved(team, task):
    """Stop as soon as we see 'APPROVED'"""
    
    async for event in team.run_stream(task=task):
        if event["type"] == "message_created":
            msg = event["message"]
            if "APPROVED" in msg.content:
                print("Approved! Stopping early.")
                break  # Exit loop
        
        elif event["type"] == "team_finished":
            return event["result"]
```

### Pattern: Progress Tracking

```python
async def run_with_progress(team, task):
    """Track progress percentage"""
    
    message_count = 0
    expected_messages = 10  # Estimate
    
    async for event in team.run_stream(task=task):
        if event["type"] == "message_created":
            message_count += 1
            progress = min(100, int(message_count / expected_messages * 100))
            print(f"Progress: {progress}%")
        
        elif event["type"] == "team_finished":
            print("Complete!")
            return event["result"]
```

---

## Common Mistakes

### Mistake 1: Not Awaiting run()
```python
# WRONG
result = team.run(task="...")  # Missing await!

# RIGHT
result = await team.run(task="...")
```

### Mistake 2: Not Awaiting Stream
```python
# WRONG
for event in team.run_stream(task="..."):  # Missing async!
    print(event)

# RIGHT
async for event in team.run_stream(task="..."):
    print(event)
```

### Mistake 3: Accessing Result Before Complete

```python
# WRONG
async for event in team.run_stream(task="..."):
    if event["type"] == "message_created":
        print(event["result"])  # Doesn't exist yet!

# RIGHT
async for event in team.run_stream(task="..."):
    if event["type"] == "team_finished":
        print(event["result"])  # Now it exists
```

### Mistake 4: Blocking Inside Async Loop

```python
# WRONG - Blocks event loop
import time
async for event in team.run_stream(task="..."):
    time.sleep(1)  # BLOCKS! ✗

# RIGHT - Use async sleep
import asyncio
async for event in team.run_stream(task="..."):
    await asyncio.sleep(1)  # Doesn't block ✓
```

---

## Next Steps

- [10-single-agent-loop.md](10-single-agent-loop.md) - Agent execution details
- [31-runtime-outputs-objects.md](31-runtime-outputs-objects.md) - TaskResult structure
- [21-observability.md](21-observability.md) - Event analysis

---

## References

- Official AutoGen: https://microsoft.github.io/autogen/
