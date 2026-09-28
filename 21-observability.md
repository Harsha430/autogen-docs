# Part 21: Observability - Logging & Monitoring

## What It Is

**Observability** lets you see what agents are doing: logging, metrics, tracing, dashboards.

```
Agent runs
  ↓
Events generated
  ↓
Logged to file/service
  ↓
Analyzed in dashboard
```

---

## What to Log

| Category | Example | Why |
|----------|---------|-----|
| **Agent activity** | "Agent A started turn 5" | Track flow |
| **Decisions** | "Chose action: search" | Debug choices |
| **Tool usage** | "Called search_web(query=...)" | Audit tools |
| **Errors** | "Tool failed: timeout" | Find bugs |
| **Metrics** | "Tokens used: 1000" | Monitor costs |

---

## Logging Strategies

### Python Logging (Standard)

```python
import logging

# Configure
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('agent.log'),
        logging.StreamHandler()  # Also print to console
    ]
)

logger = logging.getLogger(__name__)

# Use
async def run_team():
    logger.info("Starting team execution")
    result = await team.run(task="...")
    logger.info(f"Completed: {result.stop_reason}")
    return result
```

### Structured Logging (JSON)

```python
import json
from datetime import datetime

def log_structured(event_type: str, data: dict):
    """Log as JSON (searchable)"""
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "type": event_type,
        **data  # Spread additional data
    }
    print(json.dumps(log_entry))

# Usage
log_structured("agent_started", {
    "agent": "analyzer",
    "turn": 5,
    "task": "Analyze data"
})

log_structured("tool_called", {
    "tool": "search_web",
    "args": {"query": "Python"},
    "duration_ms": 1200
})
```

---

## Stream Events for Observability

```python
async def monitor_execution(team, task):
    """Stream events with logging"""
    
    logger = logging.getLogger("team_monitor")
    event_count = 0
    
    async for event in team.run_stream(task=task):
        event_count += 1
        event_type = event.get("type")
        
        if event_type == "agent_starting":
            logger.info(f"Agent starting: {event['agent']}")
        
        elif event_type == "message_created":
            msg = event["message"]
            logger.info(f"{msg.source}: {msg.content[:100]}...")
        
        elif event_type == "tool_call_starting":
            logger.info(f"Tool: {event['tool']}")
        
        elif event_type == "tool_call_completed":
            logger.info(f"Tool result: {event['result'][:100]}...")
        
        elif event_type == "team_finished":
            logger.info(f"Team finished: {event['result'].stop_reason}")
            logger.info(f"Total events: {event_count}")
            return event["result"]
```

---

## Event Analysis

### Count Message Types

```python
from collections import Counter

def analyze_messages(result):
    """Analyze execution"""
    
    message_types = Counter(type(m).__name__ for m in result.messages)
    speakers = Counter(m.source for m in result.messages)
    
    print(f"Message types: {message_types}")
    print(f"Speaker distribution: {speakers}")
    
    # Who talked most?
    top_speaker = speakers.most_common(1)[0]
    print(f"Most active: {top_speaker[0]} ({top_speaker[1]} messages)")
```

### Track Metrics

```python
def calculate_metrics(result):
    """Extract performance metrics"""
    
    metrics = {
        "total_messages": len(result.messages),
        "unique_agents": len(set(m.source for m in result.messages)),
        "stop_reason": result.stop_reason,
        "avg_message_length": sum(len(m.content) for m in result.messages) // len(result.messages)
    }
    
    return metrics

# Log metrics
metrics = calculate_metrics(result)
logger.info(f"Execution metrics: {metrics}")
```

---

## Distributed Tracing

### Trace Handoffs

```python
def trace_handoffs(result):
    """Show agent → agent transfers"""
    
    handoffs = []
    for msg in result.messages:
        if isinstance(msg, HandoffMessage):
            handoffs.append({
                "from": msg.source,
                "to": msg.to,
                "content": msg.content[:50]
            })
    
    print("Handoff chain:")
    for h in handoffs:
        print(f"  {h['from']} → {h['to']}: {h['content']}")
    
    return handoffs
```

### Flow Diagram (Text)

```python
def show_flow(result):
    """ASCII flow diagram"""
    
    agents = []
    for msg in result.messages:
        if msg.source not in agents:
            agents.append(msg.source)
    
    print(" → ".join(agents))
    # Output: agent_a → agent_b → agent_c → agent_a
```

---

## Dashboard Concepts

### Real-Time Dashboard (Pseudo-code)

```python
# In web framework (e.g., FastAPI + WebSocket)
async def stream_to_dashboard(team, task):
    """Send events to web dashboard in real-time"""
    
    async for event in team.run_stream(task=task):
        # Broadcast to all connected clients
        await broadcast_to_dashboard({
            "timestamp": time.time(),
            "event_type": event.get("type"),
            "agent": event.get("agent"),
            "content": str(event.get("message", ""))[:200]
        })
```

### Metrics Dashboard

```python
# Key metrics to display
metrics_to_track = {
    "Messages per minute": "calculate_message_rate()",
    "Token usage": "calculate_tokens()",
    "Agent utilization": "agents_speaking / total_agents",
    "Error rate": "failed_tools / total_tools",
    "Current turn": "turn_number"
}
```

---

## Debugging with Logs

### Find Errors
```python
def extract_errors(log_file):
    """Find all errors in logs"""
    errors = []
    with open(log_file, "r") as f:
        for line in f:
            if "ERROR" in line or "error" in line:
                errors.append(line.strip())
    return errors

errors = extract_errors("agent.log")
for e in errors:
    print(e)
```

### Trace Execution Path

```python
def trace_path(log_file):
    """Show agent → agent path"""
    
    path = []
    with open(log_file, "r") as f:
        for line in f:
            if "Agent" in line and "starting" in line:
                # Extract agent name
                agent = line.split("Agent ")[1].split(" ")[0]
                path.append(agent)
    
    print(" → ".join(path))
```

---

## Monitoring Tools

### Log Aggregation

```python
# Send logs to centralized service (e.g., ELK stack)
from logging.handlers import SysLogHandler

handler = SysLogHandler(address=('logs.example.com', 514))
logging.getLogger().addHandler(handler)
```

### Metrics Collection

```python
# Send metrics to service (e.g., Prometheus)
from prometheus_client import Counter, Histogram

agent_executions = Counter('agent_executions_total', 'Total agent executions')
execution_duration = Histogram('agent_duration_seconds', 'Execution time')

# Use
agent_executions.inc()
with execution_duration.time():
    result = await team.run(...)
```

---

## Common Mistakes

### Mistake 1: Logging Everything
```python
# WRONG - Too verbose
logger.debug(f"Message: {msg.content}")  # Every message!
logger.debug(f"Turn: {i}")  # Every turn!

# RIGHT - Log key events
logger.info(f"Agent {agent} completed task")
logger.error(f"Tool failed: {error}")
```

### Mistake 2: Unstructured Logging
```python
# WRONG - Hard to parse
logger.info(f"Completed: {result.stop_reason} with {len(result.messages)} messages")

# RIGHT - Structured
logger.info("execution_completed", extra={
    "stop_reason": result.stop_reason,
    "message_count": len(result.messages)
})
```

### Mistake 3: No Log Rotation
```python
# WRONG - Log file grows forever
logging.basicConfig(filename='agent.log')

# RIGHT - Rotate logs
from logging.handlers import RotatingFileHandler
handler = RotatingFileHandler('agent.log', maxBytes=1000000, backupCount=5)
logging.getLogger().addHandler(handler)
```

---

## Next Steps

- [08-run-and-streaming.md](08-run-and-streaming.md) - Event streaming
- [22-performance-cost.md](22-performance-cost.md) - Cost monitoring
- [25-complete-project.md](25-complete-project.md) - Full observability example

---

## References

- Official AutoGen: https://microsoft.github.io/autogen/
