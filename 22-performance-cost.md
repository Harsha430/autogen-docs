# Part 22: Performance & Cost Analysis

## What It Is

Understanding execution cost and speed helps optimize systems for production.

```
Run 1: 100 API calls × $0.01/1K tokens = $0.001
Run 2: 200 API calls × $0.01/1K tokens = $0.002  ← 2x cost
Problem: Unbounded API calls
Solution: Optimize to 50 calls = $0.0005
```

---

## Execution Time Measurement

### Simple Timing

```python
import time

async def measure_execution():
    """Measure total time"""
    
    start = time.time()
    result = await team.run(task="...")
    elapsed = time.time() - start
    
    print(f"Executed in {elapsed:.1f} seconds")
    return elapsed

# Usage
time_taken = await measure_execution()
```

### Per-Step Timing

```python
async def measure_steps(team, task):
    """Time each event"""
    
    start = time.time()
    events = []
    
    async for event in team.run_stream(task=task):
        elapsed = time.time() - start
        events.append({
            "type": event.get("type"),
            "time": elapsed,
            "agent": event.get("agent")
        })
    
    # Show timeline
    for e in events:
        print(f"[{e['time']:.1f}s] {e['type']}: {e['agent']}")
```

---

## Token Usage Tracking

### Estimate Token Cost

```python
# Tokens per 1K for major providers
COSTS = {
    "gpt-4": {"input": 0.03, "output": 0.06},
    "gpt-4-turbo": {"input": 0.01, "output": 0.03},
    "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015},
    "claude-3-sonnet": {"input": 0.003, "output": 0.015},
}

def estimate_cost(input_tokens, output_tokens, model="gpt-4"):
    """Calculate API cost"""
    
    rates = COSTS.get(model, COSTS["gpt-4"])
    
    input_cost = (input_tokens / 1000) * rates["input"]
    output_cost = (output_tokens / 1000) * rates["output"]
    
    total = input_cost + output_cost
    
    return {
        "input_cost": input_cost,
        "output_cost": output_cost,
        "total_cost": total
    }

# Usage
cost = estimate_cost(input_tokens=10000, output_tokens=2000, model="gpt-4")
print(f"Cost: ${cost['total_cost']:.4f}")
```

### Track Token Usage Per Agent

```python
class TokenTracker:
    """Track tokens by agent"""
    
    def __init__(self):
        self.tokens = {}  # {agent: {input: X, output: Y}}
    
    def log_tokens(self, agent: str, input_tokens: int, output_tokens: int):
        """Record token usage"""
        if agent not in self.tokens:
            self.tokens[agent] = {"input": 0, "output": 0}
        
        self.tokens[agent]["input"] += input_tokens
        self.tokens[agent]["output"] += output_tokens
    
    def get_cost(self, model="gpt-4"):
        """Calculate total cost"""
        rates = COSTS[model]
        total = 0
        
        for agent, usage in self.tokens.items():
            agent_cost = (usage["input"] / 1000) * rates["input"]
            agent_cost += (usage["output"] / 1000) * rates["output"]
            print(f"{agent}: ${agent_cost:.4f}")
            total += agent_cost
        
        return total

# Usage
tracker = TokenTracker()
tracker.log_tokens("analyzer", input_tokens=5000, output_tokens=1000)
tracker.log_tokens("reviewer", input_tokens=3000, output_tokens=500)
total_cost = tracker.get_cost()
print(f"Total cost: ${total_cost:.4f}")
```

---

## Performance Profiling

### Find Bottlenecks

```python
import cProfile
import pstats
from io import StringIO

async def profile_execution():
    """Profile CPU usage"""
    
    profiler = cProfile.Profile()
    profiler.enable()
    
    result = await team.run(task="...")
    
    profiler.disable()
    
    # Print stats
    s = StringIO()
    ps = pstats.Stats(profiler, stream=s).sort_stats('cumulative')
    ps.print_stats(10)  # Top 10
    print(s.getvalue())
```

### Memory Usage

```python
import tracemalloc

async def profile_memory():
    """Profile memory usage"""
    
    tracemalloc.start()
    
    result = await team.run(task="...")
    
    current, peak = tracemalloc.get_traced_memory()
    print(f"Current: {current / 1024 / 1024:.1f} MB")
    print(f"Peak: {peak / 1024 / 1024:.1f} MB")
```

---

## Optimization Strategies

### Reduce API Calls

```python
# WRONG - Model makes decision each turn
team = SelectorGroupChat(participants=[A, B])
# Calls model for each turn (50+ API calls)

# BETTER - Fixed routing
team = RoundRobinGroupChat(participants=[A, B])
# 0 extra calls, just agent calls
```

### Batch Operations

```python
# WRONG - Sequential
for task in tasks:
    result = await agent.run(task=task)

# BETTER - Parallel when independent
results = await asyncio.gather(*[
    agent.run(task=task) for task in tasks
])
```

### Cache Tool Results

```python
# WRONG - Same tool call twice
result1 = search_tool("Python")
result2 = search_tool("Python")  # Same call!

# BETTER - Cache
cache = {}

def cached_search(query):
    if query not in cache:
        cache[query] = search_tool(query)
    return cache[query]

result1 = cached_search("Python")
result2 = cached_search("Python")  # From cache
```

---

## Cost Optimization Patterns

### Cost Tracking Template

```python
class CostOptimizer:
    """Track and optimize costs"""
    
    def __init__(self, budget_per_task=0.10):
        self.budget = budget_per_task
        self.spent = 0
    
    async def run_with_budget(self, team, task):
        """Run task with budget limit"""
        
        result = await team.run(task=task)
        
        # Estimate cost
        tokens = calculate_tokens(result.messages)
        cost = (tokens / 1000) * 0.01  # Rough estimate
        
        self.spent += cost
        
        if self.spent > self.budget:
            print(f"⚠️ WARNING: Over budget! ${self.spent:.4f} > ${self.budget:.4f}")
        
        return result
```

### Tiered Model Selection

```python
# Use cheap model for drafts, expensive for review
async def draft_and_review(content):
    """Draft with cheap model, review with expensive"""
    
    # Draft with GPT-3.5 (cheap)
    draft_client = OpenAIClient(model="gpt-3.5-turbo")
    draft = await drafting_agent.run(task=content, client=draft_client)
    
    # Review with GPT-4 (expensive, but necessary)
    review_client = OpenAIClient(model="gpt-4")
    review = await review_agent.run(task=draft, client=review_client)
    
    return review
```

---

## Cost vs Quality Trade-offs

| Model | Cost | Quality | Speed |
|-------|------|---------|-------|
| GPT-3.5-turbo | $$ | Medium | ⚡ |
| GPT-4-turbo | $$$$ | High | 🐢 |
| Claude Sonnet | $$$ | High | ⚡ |

**Decision framework**:
- **Draft/research**: Use cheap (GPT-3.5)
- **Final output**: Use better (GPT-4, Claude)
- **High volume**: Optimize for throughput
- **High quality**: Optimize for accuracy

---

## Monitoring Cost

```python
import logging

logger = logging.getLogger("cost_monitor")

def log_cost(task_name, tokens_in, tokens_out, model):
    """Log cost metrics"""
    rates = COSTS[model]
    cost = (tokens_in / 1000) * rates["input"]
    cost += (tokens_out / 1000) * rates["output"]
    
    logger.info(f"Task: {task_name}, Cost: ${cost:.4f}, Model: {model}")

# Usage
log_cost("analyze_report", 5000, 1000, "gpt-4")
```

---

## Common Mistakes

### Mistake 1: Ignoring Cost
```python
# WRONG - No cost tracking
for i in range(1000):
    result = await team.run(task=...)  # Could be $100+!

# RIGHT - Track cost
for i in range(1000):
    result = await team.run(task=...)
    cost = estimate_cost(...)
    if total_cost > budget:
        break
```

### Mistake 2: Premature Optimization
```python
# WRONG - Optimize before profiling
# Just assume streaming is best

# RIGHT - Measure first
time_normal = await measure_execution()  # run()
time_stream = await measure_execution_stream()  # run_stream()
# Then optimize based on results
```

### Mistake 3: Not Batching

```python
# WRONG - Sequential
for i in range(100):
    result = await agent.run(task)  # 100x slower

# RIGHT - Batch if possible
results = await asyncio.gather(*[
    agent.run(task) for _ in range(100)
])
```

---

## Next Steps

- [21-observability.md](21-observability.md) - Monitoring
- [23-autogen-core.md](23-autogen-core.md) - Low-level runtime
- [25-complete-project.md](25-complete-project.md) - Full production example

---

## References

- OpenAI pricing: https://openai.com/pricing
- Anthropic pricing: https://www.anthropic.com/pricing
- Official AutoGen: https://microsoft.github.io/autogen/
