# Part 10: The Single Agent Loop - Complete Execution Flow

## High-Level Overview

A single AssistantAgent operates in a loop:

```
Task Input
    ↓
┌───────────────────────────────┐
│ Agent Loop Iteration          │
├───────────────────────────────┤
│ 1. Build context              │
│ 2. Call LLM                   │
│ 3. Parse response             │
│ 4. Tool call?                 │
│    ├─ YES: Execute tool       │
│    └─ NO: Return response     │
│ 5. Check max_tool_iterations  │
└───────────────────────────────┘
    │ (if tool called & not at max)
    ↓ (continue loop)
    │ (if tool call OR response without tool)
    ↓ (if at max iterations or no tool call)
Final Response
```

---

## Step 1: Task Acceptance

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.messages import TextMessage
import asyncio

def web_search(query: str) -> str:
    """Search the web"""
    return f"Found info about {query}"

agent = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[web_search],
    max_tool_iterations=3
)

# Trigger execution
async def main():
    result = await agent.run(
        task="Research the latest AI developments"
    )
    
    # result.messages contains the conversation
    # result.stop_reason indicates why it stopped

asyncio.run(main())
```

**What happens**:
- User calls `agent.run(task="...")`
- Agent receives TextMessage with task
- Agent's `on_messages()` method is called
- Execution loop begins

---

## Step 2: Context Building

The agent builds a context for the LLM:

```
┌─────────────────────────────────────┐
│ Context Built by Agent              │
├─────────────────────────────────────┤
│                                     │
│ 1. System Message                  │
│    "You are a helpful assistant"   │
│                                     │
│ 2. Conversation History             │
│    [All previous messages]          │
│                                     │
│ 3. Current Task                     │
│    "Research latest AI developments│
│                                     │
│ 4. Tool Schema                      │
│    [Available tools descriptions]   │
│                                     │
└─────────────────────────────────────┘
```

**In code (conceptual)**:

```python
# Agent builds messages for LLM
messages = [
    {
        "role": "system",
        "content": agent.system_message
    },
    {
        "role": "user",
        "content": "Research the latest AI developments"
    }
]

# Tool schemas
tools = [
    {
        "name": "web_search",
        "description": "Search the web",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search query"
                }
            }
        }
    }
]
```

---

## Step 3: LLM Call

```python
# Agent sends to model_client
response = await model_client.create(
    model="gpt-4",
    messages=messages,
    tools=tools,
    temperature=0.7,
    max_tokens=2048
)

# LLM processes and responds
```

**LLM sees**:
- Task: "Research latest AI developments"
- Available tools: web_search
- Role: "helpful assistant"

**LLM reasoning**:
```
"I need to research AI developments.
I have web_search available.
I should use it to find current information."
```

---

## Step 4: Parse LLM Response

LLM returns one of:

```
Option A: Tool Call Request
{
  "type": "tool_call",
  "name": "web_search",
  "arguments": {
    "query": "latest AI developments 2024"
  }
}

Option B: Text Response (No Tool)
{
  "type": "text",
  "content": "Based on my knowledge..."
}

Option C: Multiple Tool Calls
{
  "type": "tool_calls",
  "calls": [
    {"name": "web_search", "args": {...}},
    {"name": "analyze", "args": {...}}
  ]
}
```

---

## Step 5: Decision Point - Tool Call?

### Branch A: No Tool Call

```
LLM Response:
"Here's what I know about AI: [detailed response]"

Agent Decision:
"No tool call, respond with text"

Action:
Return response to user

Stop Reason: "response"
```

```python
# Agent returns directly
return TextMessage(
    content="Here's what I know about AI...",
    source="researcher"
)
```

### Branch B: Tool Call Made

```
LLM Response:
{
  "tool_calls": [{
    "name": "web_search",
    "arguments": {"query": "latest AI developments"}
  }]
}

Agent Decision:
"Tool call detected, execute it"

Action:
tool_result = web_search("latest AI developments")
```

---

## Step 6: Execute Tool (If Called)

```python
# Extract tool call
tool_name = response.tool_calls[0].name  # "web_search"
tool_args = response.tool_calls[0].arguments  # {"query": "..."}

# Execute tool
if tool_name == "web_search":
    tool_result = web_search(**tool_args)
    
# Result: string with search information
```

**Tool execution flow**:

```
web_search("latest AI developments")
    ↓
Makes actual search API call
    ↓
Parses results
    ↓
Returns formatted string
    ↓
Agent receives: "Found: ChatGPT updates..., Claude 3..., etc."
```

---

## Step 7: Add Tool Result to Context

```python
# Agent adds to conversation history:
messages = [
    {"role": "system", "content": system_message},
    {"role": "user", "content": task},
    
    # LLM's tool call request
    {
        "role": "assistant",
        "content": None,  # No text, just tool call
        "tool_calls": [{
            "name": "web_search",
            "arguments": {"query": "..."}
        }]
    },
    
    # Tool execution result
    {
        "role": "tool",
        "name": "web_search",
        "content": "Search results about AI developments..."
    }
]

# Iteration counter
tool_iterations = 1
```

---

## Step 8: Check Constraints

### max_tool_iterations Check

```python
if tool_iterations >= max_tool_iterations:
    # Stop even if LLM wants more tools
    return_best_response()
else:
    # Continue loop
    call_llm_again()
```

**Example with max_tool_iterations = 3**:

```
Iteration 1: web_search() ✓ (1/3)
    ↓
Iteration 2: analyze_results() ✓ (2/3)
    ↓
Iteration 3: summarize() ✓ (3/3)
    ↓
Iteration 4: Would exceed max → STOP
Return results from iteration 3
```

### reflect_on_tool_use Check

```python
if reflect_on_tool_use:
    # Add extra LLM call for verification
    verification_response = await model_client.create(
        messages=messages + [{
            "role": "user",
            "content": f"Please verify your tool results"
        }]
    )
    # LLM reviews tool output for quality
```

---

## Step 9: Continue Loop or Exit

### Decision Logic

```
┌─────────────────────────────────────┐
│ Should Continue Loop?               │
├─────────────────────────────────────┤
│                                     │
│ At max iterations? → STOP           │
│                     └─ Return       │
│                                     │
│ Tool call made? → CONTINUE          │
│                  └─ Loop to LLM      │
│                                     │
│ No tool call? → STOP                │
│                 └─ Return response  │
│                                     │
└─────────────────────────────────────┘
```

**In pseudocode**:

```python
tool_iterations = 0
max_tool_iterations = 3

while True:
    # Call LLM
    response = await model_client.create(...)
    
    # Check if tool call
    if has_tool_calls(response):
        tool_iterations += 1
        
        # Execute tools
        execute_tools(response.tool_calls)
        
        # Check limit
        if tool_iterations >= max_tool_iterations:
            break  # Stop loop
        
        # Otherwise continue
    else:
        # No tool call = final response
        break

return response
```

---

## Complete Example Flow

### Scenario: Research Agent

```python
# Setup
researcher = AssistantAgent(
    name="researcher",
    model_client=OpenAIChatCompletionClient(model="gpt-4"),
    tools=[web_search, parse_document],
    max_tool_iterations=3,
    system_message="""You are a research expert.
    Use web_search to find information.
    Parse documents to extract data.
    Provide thorough, sourced responses."""
)

# Task
task = "Research Python 3.13 new features"

# Execution timeline
```

#### Iteration 1: Initial Search

```
INPUT: Task = "Research Python 3.13 new features"

Context:
├─ System: "You are a research expert..."
├─ User: "Research Python 3.13 new features"
└─ Tools: [web_search, parse_document]

LLM Reasoning:
"I need to find information about Python 3.13.
I should use web_search to find this."

LLM Response:
{
  "tool_calls": [{
    "name": "web_search",
    "arguments": {"query": "Python 3.13 new features"}
  }]
}

Tool Execution:
web_search("Python 3.13 new features")
→ Returns search results about Python 3.13

Iteration Count: 1/3
```

#### Iteration 2: Parse Results

```
Context Updated:
├─ System: "You are a research expert..."
├─ User: "Research Python 3.13 new features"
├─ LLM tool call: web_search
└─ Tool result: "Found articles about..."

LLM Reasoning:
"I got search results.
Let me parse them to extract key features."

LLM Response:
{
  "tool_calls": [{
    "name": "parse_document",
    "arguments": {"content": "articles about Python 3.13..."}
  }]
}

Tool Execution:
parse_document(content)
→ Extracts: ["asyncio improvements", "error messages", "perf"]

Iteration Count: 2/3
```

#### Iteration 3: Compile Results

```
Context Updated:
├─ System: "..."
├─ User: "..."
├─ Tool call 1: web_search
├─ Result 1: search results
├─ Tool call 2: parse_document
└─ Result 2: extracted features

LLM Reasoning:
"I have the information. Time to compile response."

LLM Response:
No tool call. Text response:
"Python 3.13 introduces several improvements:
1. Asyncio improvements...
2. Better error messages...
3. Performance enhancements...
Sources: Python docs, PEPs..."

Iteration Count: 3/3
STOP - Reached max_tool_iterations
```

#### Final Output

```
Result:
"Python 3.13 introduces several improvements:
1. Asyncio improvements...
2. Better error messages...  
3. Performance enhancements...
Sources: Python docs, PEPs..."

Metadata:
├─ stop_reason: "max_tool_iterations"
├─ tool_calls: 2
└─ total_messages: 8 (system + user + 2 calls + 2 results + final)
```

---

## Preventing Infinite Loops

### Problem: Infinite Tool Calling

```python
# WRONG - Can loop forever
agent = AssistantAgent(
    name="agent",
    model_client=model_client,
    tools=[search, analyze, search, analyze...],
    # No max_tool_iterations!
)

# Risk: Agent keeps calling tools without stopping
# Example: search → analyze → search → analyze → search...
```

### Solution: Always Set max_tool_iterations

```python
# CORRECT - Has safety limit
agent = AssistantAgent(
    name="agent",
    model_client=model_client,
    tools=[search, analyze],
    max_tool_iterations=5  # Will stop after 5 tool calls
)
```

### Solution: Termination Conditions

```python
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.task import Task
from autogen_agentchat.termination import MaxMessageTermination

# Stop after 20 messages total
termination = MaxMessageTermination(max_messages=20)

team = RoundRobinGroupChat(
    participants=[agent1, agent2],
    termination_condition=termination
)
```

---

## Monitoring Tool Iterations

```python
async def research_with_monitoring(agent, task):
    """Run with iteration tracking"""
    
    iteration = 0
    max_iterations = 10
    
    # In real code, use run_stream for visibility
    result = await agent.run(task=task)
    
    # Check result metadata
    final_message = result.messages[-1]
    
    print(f"Completed: {final_message.content}")
    print(f"Stop reason: {result.stop_reason if hasattr(result, 'stop_reason') else 'unknown'}")
    
    return result
```

### Using run_stream for Visibility

```python
async def research_with_streaming(agent, task):
    """Stream events to see iterations"""
    
    iteration = 0
    
    async for event in agent.run_stream(task=task):
        iteration += 1
        print(f"Iteration {iteration}:")
        print(f"  Type: {type(event).__name__}")
        print(f"  Content: {str(event)[:100]}...")
        
        # Limit iterations
        if iteration >= 10:
            print("Reached iteration limit")
            break
    
    print(f"Total iterations: {iteration}")
```

---

## Performance Considerations

### Token Usage

Each iteration increases token usage:

```
Iteration 1:
Input tokens: ~200 (task + tools + system message)
Output tokens: ~100 (tool call JSON)
Total: ~300

Iteration 2:
Input tokens: ~300 (add previous + tool result)
Output tokens: ~150
Total: ~450

Iteration 3:
Input tokens: ~450
Output tokens: ~200
Total: ~650

Total for 3 iterations: ~1400 tokens
```

### Cost Optimization

```python
# For expensive models, reduce iterations
agent = AssistantAgent(
    max_tool_iterations=3  # Fewer iterations = lower cost
)

# For cheaper models, allow more
agent = AssistantAgent(
    max_tool_iterations=10  # More comprehensive research
)

# Use reflect_on_tool_use sparingly
agent = AssistantAgent(
    reflect_on_tool_use=True,  # Adds ~1-2 LLM calls per tool
    max_tool_iterations=3  # Limit to compensate
)
```

---

## Debugging Tool Iterations

```python
async def debug_iterations(agent, task):
    """Debug tool iteration flow"""
    
    async for event in agent.run_stream(task=task):
        print(f"\nEvent: {type(event).__name__}")
        
        # Tool call event
        if "tool" in str(type(event)).lower():
            print(f"  Tool called: {event}")
        
        # Message event
        elif "message" in str(type(event)).lower():
            print(f"  Message: {event.content[:100]}...")
        
        # Other events
        else:
            print(f"  {event}")
```

---

## Common Issues & Fixes

### Issue 1: Agent Stops Too Early

```python
# WRONG - Stops after 1 tool call
agent = AssistantAgent(max_tool_iterations=1)

# Task needs 3 tool calls but stops at 1
# Result is incomplete

# FIX
agent = AssistantAgent(max_tool_iterations=10)
```

### Issue 2: Agent Loops Infinitely

```python
# WRONG - No limit set
agent = AssistantAgent(
    # max_tool_iterations not set
    # Defaults to 10, which might still be too high
)

# FIX - Explicitly set and monitor
agent = AssistantAgent(
    max_tool_iterations=5,
    reflect_on_tool_use=False  # Reduce LLM calls
)
```

### Issue 3: High Token Usage

```python
# WRONG - Many iterations with reflection
agent = AssistantAgent(
    max_tool_iterations=20,
    reflect_on_tool_use=True  # +1-2 LLM calls per tool
)

# Cost: ~20 tools + 20 reflections = 40 LLM calls!

# FIX
agent = AssistantAgent(
    max_tool_iterations=5,  # Fewer iterations
    reflect_on_tool_use=False  # Only when needed
)
```

---

## Understanding What You Receive: TaskResult

After execution completes, you get a **TaskResult** object. To understand its structure:

**See [31-runtime-outputs-objects.md](31-runtime-outputs-objects.md)** for:
- Exact fields in TaskResult
- How to access messages, stop_reason, metadata
- How to safely extract information
- Real examples of inspecting outputs

**Quick example**:
```python
result = await agent.run(task="Research Python")

# TaskResult contains:
print(result.messages)      # List of all messages
print(result.stop_reason)   # Why it stopped
print(result.metadata)      # Execution details
```

---

## Next Steps

Now that you understand:
- ✅ Complete agent loop execution
- ✅ Tool iteration flow
- ✅ How max_tool_iterations works
- ✅ Preventing infinite loops
- ✅ Performance optimization
- ✅ **NEW**: What TaskResult contains

Move to:
- **[31-runtime-outputs-objects.md](31-runtime-outputs-objects.md)** to understand output structures
- **[11-teams.md](11-teams.md)** to coordinate multiple agents
- **[08-run-and-streaming.md](08-run-and-streaming.md)** to master async execution
- **[21-observability.md](21-observability.md)** to monitor execution

---

## References

- [AutoGen Agent Execution Documentation](https://microsoft.github.io/autogen/)
- [Tool Calling in LLMs](https://platform.openai.com/docs/guides/function-calling)

---

## Version Notes

- **Current Version**: AutoGen 0.2.x
- **Tool Iterations**: Controlled by max_tool_iterations parameter
- **Reflection**: reflect_on_tool_use adds extra LLM calls
