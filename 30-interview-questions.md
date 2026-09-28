# Part 30: Interview Questions & Answers

## Level 1: Fundamentals

### Q1: What is AutoGen?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
> AutoGen is a framework for building multi-agent systems where autonomous agents collaborate through message-based communication. Agents can use LLMs for reasoning, call tools for execution, and communicate with other agents. It solves the problem of coordinating multiple specialized AI systems working together on complex tasks.

**Key Points to Cover**:
- Message-based communication
- Multiple specialized agents
- LLM + tool combination
- Collaborative problem solving

**Follow-up Questions They Might Ask**:
- What problem does AutoGen solve?
- How is it different from just calling an LLM API?
- When would you use AutoGen vs a simple LLM call?

---

### Q2: What is the difference between an Agent and an LLM?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> An LLM (like GPT-4) is a text generation model. It processes text and generates text. An Agent is a system that uses an LLM as one component. An agent:
> - Maintains conversation history
> - Decides what to do (call tool? call other agent? respond?)
> - Manages state and execution flow
> - Coordinates with other agents
>
> The LLM only generates text. The agent makes decisions about what to do next.

**Visual**: 
```
LLM: "text in" → "text out" (stateless)
Agent: "task" → (decision-making) → LLM → (more decisions) → "result" (stateful)
```

---

### Q3: Explain the purpose of a Tool in AutoGen.
**Expected Answer Length**: 1 minute

**Sample Answer**:
> A tool is a Python function that an agent can call. Tools extend agent capabilities beyond text generation. When an agent needs to do something (search the web, execute code, query a database), it calls a tool. 
>
> The flow is:
> 1. Agent's LLM decides "I need web_search"
> 2. Agent executes the tool
> 3. Tool returns result (string)
> 4. LLM processes result and continues

**Key Point**: Tools are execution capabilities, not decision-making.

---

### Q4: What is a Model Client and why does it exist?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> A Model Client is an abstraction layer that separates agents from LLM provider details. 
>
> Without it:
> - Agents are coupled to OpenAI API details
> - Switching providers requires rewriting agents
> - Hard to test with mocks
>
> With it:
> - Agents use generic interface (create(), stream(), etc)
> - Swap OpenAI for Azure with one config change
> - Easy to mock for testing
>
> This follows the dependency inversion principle: agents depend on abstraction, not concrete implementations.

---

### Q5: What is the purpose of max_tool_iterations?
**Expected Answer Length**: 1 minute

**Sample Answer**:
> `max_tool_iterations` is a safety mechanism that prevents infinite tool-calling loops. It limits how many times an agent can call tools before returning a response.
>
> Example: If max_tool_iterations = 3:
> - Tool call 1: search ✓
> - Tool call 2: analyze ✓
> - Tool call 3: summarize ✓
> - Tool call 4: Would exceed → STOP
>
> This prevents scenarios where an agent keeps calling tools without ever providing a final answer.

---

### Q6: What does reflect_on_tool_use do?
**Expected Answer Length**: 1 minute

**Sample Answer**:
> When `reflect_on_tool_use=True`, the agent adds a verification step after each tool call. The LLM reviews tool results and validates them.
>
> Without:
> Tool call → Result → Continue
>
> With:
> Tool call → Result → LLM: "Is this correct?" → Continue
>
> Trade-off: Better quality but costs more tokens (extra LLM call).

---

### Q7: What are the three layers of AutoGen architecture?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```
1. AgentChat (High-level)
   - What you use for most applications
   - Agents, Teams, Messages, Tools
   
2. AutoGen Core (Low-level)
   - Event-driven runtime
   - Direct agent-to-agent RPC
   - Advanced routing
   
3. Model Clients (Abstraction)
   - LLM provider independence
   - OpenAI, Azure, custom implementations
```

**Key Point**: You use AgentChat 95% of the time. Core is for special cases.

---

### Q8: What's the difference between State, Memory, and History?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
> These are three different things:
>
> **State**: Current execution information
> - "Agent is at iteration 2 of 5"
> - "Processing task: research"
> - Temporary, specific to current execution
>
> **History**: Chronological record of all interactions
> - "Message 1: User asked X"
> - "Message 2: Agent responded Y"
> - Complete conversation log
>
> **Memory**: Useful information retained for future
> - "Python has 3.13 version (learned in task 1)"
> - Used in task 2: "What's new in Python 3.13?"
> - Persistent, learned knowledge

---

### Q9: What is a TextMessage?
**Expected Answer Length**: 1 minute

**Sample Answer**:
> A TextMessage is how agents communicate in AutoGen. It contains:
> - content: The actual message text
> - source: Who sent it (agent name)
> - timestamp: When it was sent
>
> Example:
> ```python
> TextMessage(
>   content="Here's what I found...",
>   source="researcher"
> )
> ```

---

### Q10: When would you use AutoGen vs just calling an LLM API directly?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
> **Use LLM API directly** when:
> - Simple single task
> - No tool calls needed
> - No multi-agent coordination required
> - Speed is critical
>
> **Use AutoGen** when:
> - Multiple tool calls in sequence
> - Multiple agents with different roles
> - Approval/human-in-loop workflows
> - Complex reasoning chains
> - Need state persistence
> - Need observability/debugging
>
> Analogy: Calling LLM API = ordering a taxi. Using AutoGen = having a team of coordinators managing a complex project.

---

## Level 2: Implementation

### Q11: How does the single-agent execution loop work?
**Expected Answer Length**: 3 minutes

**Sample Answer**:
```
1. Agent receives task
2. Build context (history + tools + system message)
3. Call LLM with context
4. LLM responds with: tool call OR text response
5. If tool call:
   a. Extract tool name and arguments
   b. Execute tool (call Python function)
   c. Get tool result (string)
   d. Add result to context
   e. Check: at max_tool_iterations?
      - YES: Return current state
      - NO: Go to step 3 (call LLM again)
6. If text response:
   Return response to user
```

**Key Point**: Tool calls create a loop where LLM and tool execution alternate.

---

### Q12: How do you create a tool that an agent can use?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
from typing import Annotated

def web_search(
    query: Annotated[str, "Search query string"],
    num_results: Annotated[int, "Number of results"] = 5
) -> str:
    """Search the web for information.
    
    Use this tool when you need current information.
    """
    # Implementation
    return "Search results..."

# Pass to agent
agent = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[web_search]
)
```

**Requirements**:
- Python function with type hints
- Return type must be str
- Include docstring
- Use Annotated for parameter descriptions

---

### Q13: What's the difference between run() and run_stream()?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
| Aspect | run() | run_stream() |
|--------|-------|-------------|
| Returns | TaskResult (final) | AsyncIterator (events) |
| Waits for completion? | Yes | No |
| Can see intermediate events? | No | Yes |
| Use case | Simple tasks | Debugging, observability |

**Example**:
```python
# run() - wait for complete result
result = await agent.run(task="...")
print(result.messages[-1].content)

# run_stream() - see everything happening
async for event in agent.run_stream(task="..."):
    print(f"Event: {event}")
```

---

### Q14: How do you pass tools to an agent?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
```python
def tool1(param: str) -> str: ...
def tool2(param: int) -> str: ...

agent = AssistantAgent(
    name="agent",
    model_client=model_client,
    tools=[tool1, tool2]  # List of functions
)
```

**Process**:
1. Define functions with type hints
2. Pass as list to agent
3. Agent extracts function signatures
4. Agent creates JSON schemas
5. Sends schemas to LLM
6. LLM decides which tools to call
7. Agent executes when needed

---

### Q15: What happens when you set max_tool_iterations=1?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
Agent can make at most 1 tool call before returning.

**Scenario**:
- Task requires 3 searches to complete thoroughly
- max_tool_iterations=1 allows only 1 search
- After 1 tool call, agent must respond with whatever it has
- Result is incomplete

**When to use**:
- Very simple tasks that need 1 tool call
- Cost optimization (fewer LLM calls)
- Protecting against infinite loops

**Typical value**: max_tool_iterations=5-10 for complex tasks

---

### Q16: How does the LLM decide to call a tool?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
> The LLM sees:
> 1. Task: "Research Python 3.13"
> 2. Available tools: [web_search, parse_document]
> 3. Tool descriptions (schema)
> 4. System message: "Use tools when needed"
>
> LLM reasoning:
> "To research Python 3.13, I should search the web. 
>  I have web_search available. I'll call it."
>
> LLM outputs:
> ```json
> {
>   "tool_calls": [{
>     "name": "web_search",
>     "arguments": {"query": "Python 3.13 features"}
>   }]
> }
> ```

**Key Point**: LLM makes the decision based on task + available tools. It doesn't execute; only decides.

---

### Q17: What's the difference between Handoff and Tool Calling?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
| Aspect | Tool Calling | Handoff |
|--------|--------------|---------|
| **What** | Function execution | Agent transfer |
| **Purpose** | Execute capability | Delegate to specialist |
| **Who decides** | LLM (via schema) | Agent explicitly |
| **Message type** | Implicit in tool call | HandoffMessage |
| **Flow** | Execute → return result | Transfer → other agent takes over |

**Example**:
```python
# Tool calling
LLM: "I need web_search" → Execute → "Results here"

# Handoff
Agent: "This needs writer expertise" → 
      Send HandoffMessage to writer →
      Writer takes over and continues
```

---

### Q18: How do you configure OpenAI API with AutoGen?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
```python
from autogen_ext.models.openai import OpenAIChatCompletionClient

model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY"),
    temperature=0.7,
    max_tokens=2048,
    timeout=60
)

agent = AssistantAgent(
    name="agent",
    model_client=model_client
)
```

**Best Practices**:
- Never hardcode API key
- Use environment variables
- Set temperature based on task (0.2 for facts, 1.2 for creative)
- Set max_tokens to limit output
- Use timeout for safety

---

### Q19: How do you limit tool output tokens?
**Expected Answer Length**: 1 minute

**Sample Answer**:
> Via `max_tokens` parameter on model_client. This limits **output** tokens (LLM response length).
>
> ```python
> model_client = OpenAIChatCompletionClient(
>     model="gpt-4",
>     max_tokens=2048  # Response limited to 2048 tokens
> )
> ```
>
> Note: This doesn't limit tool output. Tools can return any size string. The limit only applies to LLM's response.

---

### Q20: What does TaskResult contain?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> TaskResult contains:
>
> **messages**: All messages from execution
> ```python
> result.messages[-1]  # Final message
> ```
>
> **stop_reason**: Why execution stopped
> - "response" - Agent responded
> - "max_tool_iterations" - Tool limit reached
> - "max_turns" - Team turned limit
> - "cancelled" - User cancelled
>
> **metadata**: Execution info
> - Token usage (if available)
> - Timing information
> - Other provider-specific data

---

## Level 3: System Design

### Q21: Design a production multi-agent research system.
**Expected Answer Length**: 5-7 minutes

**Sample Answer**:

**Architecture**:
```
User Query
    ↓
Orchestrator/Team
    ├── Research Agent
    │   ├── Web search tool
    │   ├── Document parser tool
    │   └── Role: Find information
    │
    ├── Analyzer Agent
    │   ├── Data analyzer tool
    │   ├── Aggregator tool
    │   └── Role: Synthesize findings
    │
    ├── Writer Agent
    │   ├── Report generator tool
    │   ├── Formatter tool
    │   └── Role: Create output
    │
    └── Reviewer Agent
        ├── Validator tool
        └── Role: Approve/revise
    ↓
Human Approver
    ├── Approve → Done
    └── Reject → Back to Research
    ↓
Final Report
```

**Components**:

1. **Researcher Agent**:
```python
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[web_search, parse_document],
    system_message="Find comprehensive information",
    max_tool_iterations=5
)
```

2. **Analyzer Agent**:
```python
analyzer = AssistantAgent(
    name="analyzer",
    model_client=model_client,
    tools=[analyze_data, summarize],
    system_message="Synthesize findings",
    handoffs=["writer"]
)
```

3. **Writer Agent**:
```python
writer = AssistantAgent(
    name="writer",
    model_client=model_client,
    tools=[generate_report, format_document],
    system_message="Create polished output",
    handoffs=["reviewer"]
)
```

4. **Team Orchestration**:
```python
team = RoundRobinGroupChat(
    participants=[researcher, analyzer, writer, human_reviewer],
    max_turns=10,
    termination_condition=MaxMessageTermination(50)
)
```

**Key Design Decisions**:
- Specialized agents with specific tools
- RoundRobin for sequential phases
- Handoffs for controlled flow
- Human reviewer as final checkpoint
- max_tool_iterations prevents infinite loops
- max_turns prevents runaway execution

---

### Q22: How would you implement Human-in-the-Loop (HITL)?
**Expected Answer Length**: 4 minutes

**Sample Answer**:

**Workflow**:
```
┌─────────────┐
│ Research   │
└──────┬──────┘
       │
┌──────▼──────────────────┐
│ Analyst Reviews         │
└──────┬──────────────────┘
       │
       ├─ HUMAN CHECKPOINT
       │  ├─ Approve → Continue to writer
       │  ├─ Revise → Back to researcher
       │  └─ Reject → Stop
       │
┌──────▼──────────────────┐
│ Write Report            │
└──────┬──────────────────┘
       │
       ├─ HUMAN CHECKPOINT
       │  ├─ Approve → Done
       │  ├─ Revise → Back to analyst
       │  └─ Reject → Stop
       │
└──────▼──────────────────┘
   Final Output
```

**Implementation**:
```python
from autogen_agentchat.agents import UserProxyAgent

# Human is part of team
human = UserProxyAgent(
    name="human_approver",
    input_func=lambda: input("Approve? (yes/no/revise): ")
)

# Team includes human
team = RoundRobinGroupChat(
    participants=[
        researcher,
        analyzer,
        human,  # Human approval checkpoint
        writer,
        human   # Another checkpoint
    ],
    max_turns=15
)

# Execution
result = await team.run(
    task="Research and analyze Python 3.13"
)
```

**Key Points**:
- UserProxyAgent represents human input
- Human added to team like any other agent
- Can have multiple human checkpoints
- Approval flow is part of orchestration
- State persists across checkpoints

---

### Q23: How would you prevent infinite loops in a team?
**Expected Answer Length**: 3 minutes

**Sample Answer**:

**Multiple Safeguards**:

1. **max_tool_iterations per agent**:
```python
agent = AssistantAgent(
    max_tool_iterations=5  # Each agent limited
)
```

2. **max_turns on team**:
```python
team = RoundRobinGroupChat(
    participants=[...],
    max_turns=10  # Team stops after 10 turns
)
```

3. **Termination condition**:
```python
from autogen_agentchat.termination import MaxMessageTermination

team = RoundRobinGroupChat(
    participants=[...],
    termination_condition=MaxMessageTermination(max_messages=100)
)
```

4. **Content-based termination**:
```python
class CustomTermination(TerminationCondition):
    def _is_terminal(self, messages):
        last = messages[-1].content
        return "FINAL ANSWER" in last or "COMPLETE" in last
```

5. **Explicit handoff logic**:
```python
# Only allow specific handoffs
researcher = AssistantAgent(
    name="researcher",
    handoffs=["analyzer"]  # Can only go to analyzer
)

analyzer = AssistantAgent(
    name="analyzer",
    handoffs=["writer"]  # Can only go to writer
)

writer = AssistantAgent(
    name="writer",
    handoffs=[]  # No handoffs (terminal)
)
```

**Best Practice**: Use multiple safeguards together.

---

### Q24: How would you monitor token usage and costs?
**Expected Answer Length**: 3 minutes

**Sample Answer**:

**Tracking Structure**:
```python
class ExecutionMetrics:
    def __init__(self):
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.total_calls = 0
        self.errors = 0
    
    @property
    def total_tokens(self):
        return self.total_input_tokens + self.total_output_tokens
    
    def estimated_cost(self, model="gpt-4"):
        # GPT-4: $0.03 per 1K input, $0.06 per 1K output
        input_cost = (self.total_input_tokens / 1000) * 0.03
        output_cost = (self.total_output_tokens / 1000) * 0.06
        return input_cost + output_cost
```

**Collection**:
```python
metrics = ExecutionMetrics()

async for event in team.run_stream(task):
    # Track each event
    if hasattr(event, 'token_count'):
        metrics.total_input_tokens += event.token_count.input
        metrics.total_output_tokens += event.token_count.output
    
    metrics.total_calls += 1

print(f"Total tokens: {metrics.total_tokens}")
print(f"Estimated cost: ${metrics.estimated_cost():.2f}")
```

**Per-Agent Metrics**:
```python
agent_metrics = {
    "researcher": {"tokens": 5000, "calls": 3},
    "analyzer": {"tokens": 2000, "calls": 2},
    "writer": {"tokens": 3000, "calls": 1},
}

for agent, m in agent_metrics.items():
    cost = (m["tokens"] / 1000) * 0.05  # Simplified
    print(f"{agent}: {m['tokens']} tokens, ${cost:.2f}")
```

---

### Q25: When would you use AutoGen Core instead of AgentChat?
**Expected Answer Length**: 2.5 minutes

**Sample Answer**:

**Use AgentChat when** (95% of cases):
- Standard multi-agent patterns
- You want high-level abstractions
- Communication is implicit (via team)
- Agents are similar (all AssistantAgent)

**Use Core when**:
1. **Complex routing logic**:
   - Agents need direct RPC (request/response)
   - Pub/sub messaging needed
   - Custom message routing

2. **Heterogeneous agents**:
   - Mix of different agent types
   - Some agents have special requirements
   - Custom lifecycle management

3. **Fine-grained control**:
   - Need low-level execution control
   - Custom event handling
   - Advanced debugging

4. **Performance optimization**:
   - Direct routing more efficient
   - Reduce intermediary layers
   - Custom optimization

**Example (Core)**:
```python
from autogen_core import AgentRuntime, TopicId, Subscription

runtime = AgentRuntime()

# Direct agent-to-agent RPC
response = await runtime.send_message(
    message="Analyze this data",
    recipient=analyzer_agent_id
)

# Pub/sub messaging
topic = TopicId(name="results")
subscription = Subscription(topic=topic, agent_id=agent_id)
```

**Reality**: Most projects don't need Core. Start with AgentChat.

---

### Q26: How would you design a multi-agent system for production deployment?
**Expected Answer Length**: 5 minutes

**Sample Answer**:

**Architecture**:
```
┌─────────────────────────────────┐
│ Input Layer                     │
│ • Queue (async job queue)       │
│ • Request validation            │
└──────────────┬──────────────────┘
               │
┌──────────────▼──────────────────┐
│ Agent Coordination Layer        │
│ • Teams                         │
│ • Termination logic             │
│ • Error handling                │
└──────────────┬──────────────────┘
               │
┌──────────────▼──────────────────┐
│ Observability Layer             │
│ • Logging                       │
│ • Metrics collection            │
│ • Distributed tracing           │
└──────────────┬──────────────────┘
               │
┌──────────────▼──────────────────┐
│ Output Layer                    │
│ • Result storage                │
│ • Notification system           │
│ • State persistence             │
└─────────────────────────────────┘
```

**Configuration**:
```python
# Model client pooling
model_clients = {
    "fast": OpenAIChatCompletionClient(model="gpt-3.5-turbo"),
    "quality": OpenAIChatCompletionClient(model="gpt-4"),
}

# Agents with error handling
agents = {
    "researcher": AssistantAgent(
        name="researcher",
        model_client=model_clients["quality"],
        tools=[web_search],
        max_tool_iterations=5
    ),
    # ... more agents
}

# Team with safety
team = RoundRobinGroupChat(
    participants=[agents["researcher"], ...],
    termination_condition=MaxMessageTermination(100),
    max_turns=20
)
```

**State Management**:
```python
# Save state for resumption
state = await team.save_state()
db.save(job_id, state)

# Resume later
state = db.load(job_id)
await team.load_state(state)
result = await team.run(task="Continue analysis")
```

**Monitoring**:
```python
# Track all executions
for result in results:
    log.info(f"Job {job_id}: {result.stop_reason}")
    
    # Metrics
    metrics.record(
        job_id=job_id,
        tokens_used=result.metadata.get('tokens'),
        duration=result.metadata.get('duration'),
        status=result.stop_reason
    )
```

---

### Q27: Compare AutoGen to LangGraph. When would you use each?
**Expected Answer Length**: 4 minutes

**Sample Answer**:

| Aspect | AutoGen | LangGraph |
|--------|---------|-----------|
| **Focus** | Multi-agent collaboration | Workflow DAGs |
| **State** | Agent-managed | Explicit state dict |
| **Routing** | Message-based | Graph edges |
| **Orchestration** | Teams (implicit) | Explicit graph |
| **Best for** | Autonomous agents | Deterministic workflows |

**AutoGen Use Cases**:
- Autonomous multi-agent teams
- Agents decide next step
- Complex collaboration
- Human-in-loop workflows

**LangGraph Use Cases**:
- Deterministic workflows
- Clear decision points
- Conditional logic
- Linear chains

**Example**:
```python
# AutoGen: Agents decide
team = RoundRobinGroupChat([researcher, analyzer, writer])
# Researchers and analyzer collaborate,
# decisions are made by agents/LLMs

# LangGraph: Graph decides
graph = StateGraph(State)
graph.add_node("researcher", research_node)
graph.add_node("analyzer", analyze_node)
graph.add_node("writer", write_node)
graph.add_edge("researcher", "analyzer")
graph.add_edge("analyzer", "writer")
# Flow is predetermined, no decisions
```

**Choose AutoGen when**: Agents should collaborate autonomously

**Choose LangGraph when**: Workflow is known and deterministic

---

### Q28: How would you handle errors in AutoGen applications?
**Expected Answer Length**: 3 minutes

**Sample Answer**:

**Error Types**:

1. **Tool Errors**:
```python
def web_search(query: str) -> str:
    try:
        results = search_api.query(query)
        return str(results)
    except NetworkError as e:
        return f"Search failed: {e}. Please try again."
    except Exception as e:
        return f"Unexpected error: {e}"
```

2. **Termination Errors**:
```python
result = await team.run(task="...")
if result.stop_reason == "max_turns":
    log.warning("Reached max turns - incomplete result")
elif result.stop_reason == "response":
    log.info("Completed normally")
```

3. **Configuration Errors**:
```python
try:
    model_client = OpenAIChatCompletionClient(
        model="gpt-4",
        api_key=os.getenv("OPENAI_API_KEY")
    )
except ValueError as e:
    log.error(f"Invalid config: {e}")
    raise
```

4. **State Errors**:
```python
try:
    await team.load_state(corrupted_state)
except StateError as e:
    log.error(f"Cannot resume: {e}")
    # Start fresh
    result = await team.run(task="...")
```

---

### Q29: How would you optimize costs in a production AutoGen system?
**Expected Answer Length**: 3 minutes

**Sample Answer**:

1. **Model Selection**:
```python
def select_model(complexity):
    if complexity == "simple":
        return OpenAIChatCompletionClient(model="gpt-3.5-turbo")
    elif complexity == "medium":
        return OpenAIChatCompletionClient(model="gpt-4-turbo")
    else:
        return OpenAIChatCompletionClient(model="gpt-4")
```

2. **Token Limits**:
```python
agent = AssistantAgent(
    model_client=model_client,
    max_tool_iterations=3,  # Fewer iterations
    reflect_on_tool_use=False  # Skip verification
)
```

3. **Caching**:
```python
# Cache tool results
cache = {}
def cached_search(query):
    if query in cache:
        return cache[query]
    result = web_search(query)
    cache[query] = result
    return result
```

4. **Batch Processing**:
```python
# Process multiple jobs in batch
# Reuse model_client connection
# Reduce API call overhead
results = []
for task in tasks:
    result = await agent.run(task=task)
    results.append(result)
```

5. **Monitoring & Limits**:
```python
max_tokens_per_hour = 1_000_000
tokens_used = 0

for result in results:
    tokens_used += result.metadata['tokens']
    if tokens_used > max_tokens_per_hour:
        log.warning("Approaching token limit")
        # Pause processing
```

---

### Q30: Describe the complete execution flow of a team with multiple agents.
**Expected Answer Length**: 5 minutes

**Sample Answer**:

**Setup**:
```python
researcher = AssistantAgent(name="researcher", ...)
analyzer = AssistantAgent(name="analyzer", ...)
writer = AssistantAgent(name="writer", ...)

team = RoundRobinGroupChat(
    participants=[researcher, analyzer, writer],
    max_turns=10
)
```

**Execution Timeline**:

```
USER INPUT: "Research Python 3.13"
    ↓

TURN 1 - RESEARCHER
├─ Receives task
├─ Calls web_search tool
├─ Gets search results
├─ Returns analysis of findings
└─ Outputs: TextMessage from researcher

TURN 2 - ANALYZER
├─ Receives researcher's message
├─ Calls aggregation tool
├─ Synthesizes findings
├─ Structures insights
└─ Outputs: TextMessage from analyzer

TURN 3 - WRITER
├─ Receives analyzer's message
├─ Calls report generator tool
├─ Formats document
├─ Polishes language
└─ Outputs: TextMessage from writer

TURN 4+ - Continues until:
├─ max_turns reached, OR
├─ Termination condition met, OR
├─ Final message is response (not tool call)

FINAL RESULT
├─ TaskResult with all messages
├─ stop_reason: why it stopped
└─ metadata: execution info
```

**Context Flow**:
```
Iteration 1: [system, user_task]
Iteration 2: [system, user_task, researcher_response]
Iteration 3: [system, user_task, researcher_response, analyzer_response]
Iteration 4: [system, user_task, researcher_response, analyzer_response, writer_response]
```

**Each Agent's Process**:
```
Agent receives message
    ↓
Builds context (system + history + available tools)
    ↓
Calls LLM with context
    ↓
LLM decides: tool call OR respond?
    ├─ Tool: Execute → return result
    └─ Respond: Return TextMessage
    ↓
Next agent in sequence
```

**Stop Conditions**:
- After writer, goes back to researcher (round-robin)
- Stops after 10 turns (max_turns)
- Or when termination condition is met

---

## Additional Level 1: Fundamentals Extended (15 more questions)

### Q31: What is the difference between RoundRobinGroupChat and SelectorGroupChat?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
| Aspect | RoundRobin | Selector |
|--------|-----------|----------|
| **Order** | Fixed (A→B→C→A) | AI-decided |
| **Routing** | Predetermined | Dynamic |
| **Cost** | Lower (no selector calls) | Higher (selector LLM calls) |
| **Predictability** | 100% predictable | Less predictable |
| **Use case** | Sequential tasks | Adaptive routing |

**When to use**:
- **RoundRobin**: Research → Analysis → Writing (clear sequence)
- **Selector**: Customer support (different agent per issue type)

---

### Q32: What is a HandoffMessage and when would you use it?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> A HandoffMessage is sent when one agent explicitly wants to transfer control to another agent.

```python
HandoffMessage(
    to="specialist_agent",
    content="This requires expertise I don't have",
    source="general_agent"
)
```

**When to use**:
- Escalation workflows
- Agent knows it's not the right specialist
- Clear delegation needed

**Difference from routing**: Agent explicitly says "hand off", vs Selector deciding.

---

### Q33: What does the system_message parameter do?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> The system_message instructs the LLM how to behave and what role to play.

**Example**:
```python
researcher = AssistantAgent(
    system_message="You are a research expert. "
                   "Find authoritative sources. "
                   "Verify all facts. "
                   "Be thorough."
)
```

**Key points**:
- Sets agent personality/role
- Guides decision-making
- Affects tool selection
- One of the most important parameters

---

### Q34: What is the difference between a TextMessage and a ToolCallSummaryMessage?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
| Type | Content | Purpose |
|------|---------|---------|
| **TextMessage** | Text response | Agent communication |
| **ToolCallSummaryMessage** | Tool call + result | Document tool execution |

**Example**:
```python
# TextMessage
TextMessage(content="Here's what I found", source="agent")

# ToolCallSummaryMessage
ToolCallSummaryMessage(
    tool_name="web_search",
    tool_call_args={"query": "Python 3.13"},
    tool_call_result="Found 100 results..."
)
```

---

### Q35: What does termination_condition do vs max_turns?
**Expected Answer Length**: 2 minutes

**Sample Answer**:

| Aspect | max_turns | termination_condition |
|--------|-----------|----------------------|
| **What** | Agent count limit | Custom logic |
| **Type** | Number | Callable |
| **Example** | max_turns=10 | stop_words=["DONE"] |
| **Triggered when** | Turns reached | Condition true |
| **Needed** | Yes (safety) | Optional |

**Example**:
```python
# Both together
team = RoundRobinGroupChat(
    participants=[...],
    max_turns=20,  # Hard safety limit
    termination_condition=StopMessageTermination(stop_words=["FINAL"])
)
# Stops if EITHER max_turns OR "FINAL" seen
```

---

### Q36: How do you know when an agent should use a tool?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> The agent (LLM) decides based on:
> 1. Available tools (schemas passed to LLM)
> 2. Task requirements
> 3. System message guidance
> 4. Conversation history

**Example**:
```
Task: "Research Python 3.13"
Available tools: [web_search, calculator, file_reader]
LLM reasoning: "To research, I need web_search"
Action: Calls web_search("Python 3.13")
```

**Key point**: LLM sees all tools and decides which (if any) to call.

---

### Q37: What is the difference between state and history?
**Expected Answer Length**: 2 minutes

**Sample Answer**:

| Concept | Scope | Lifetime | Use |
|---------|-------|----------|-----|
| **State** | Current execution | One run | "Where am I?" |
| **History** | All messages | Full conversation | "What happened?" |

**Example**:
```python
# State
{
    "current_turn": 5,
    "current_agent": "analyzer",
    "messages_count": 12
}

# History
[
    TextMessage("Research..."),
    TextMessage("Found..."),
    TextMessage("Analyzing..."),
    ...
]
```

---

### Q38: What is max_tool_iterations and why is it important?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> `max_tool_iterations` limits how many times an agent calls tools before returning a response.

**Why important**:
- Prevents infinite tool loops
- Controls cost (fewer LLM calls)
- Ensures progress

**Example**:
```python
agent = AssistantAgent(
    max_tool_iterations=5
)

# Agent can call tools: 1, 2, 3, 4, 5
# Tool call 6 → Stops and returns current response
```

---

### Q39: What is reflect_on_tool_use?
**Expected Answer Length**: 1 minute

**Sample Answer**:
> When `reflect_on_tool_use=True`, agent reflects on tool results before continuing.

**Without**:
```
Call web_search → Get results → Continue
```

**With**:
```
Call web_search → Get results → LLM: "Are these results good?" → Continue
```

**Trade-off**: Better quality but +1 LLM call per tool (more tokens, more cost).

---

### Q40: What does model_client.stream do?
**Expected Answer Length**: 1 minute

**Sample Answer**:
> `stream=True` returns tokens as they arrive instead of waiting for full response.

**Without**:
```python
response = client.create(...)  # Wait for full response
```

**With**:
```python
async for token in client.stream(...):
    print(token)  # Each token as it arrives
```

**Use case**: Real-time streaming, observability, responsiveness.

---

### Q41: What is the relationship between tool schema and tool calling?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
> Tool schema is how the LLM knows about a tool.

**Process**:
1. Define tool (Python function)
2. Extract signature
3. Create JSON schema
4. Send schema to LLM
5. LLM sees available tools
6. LLM decides to call one
7. Agent executes

**Schema includes**:
- Tool name
- Description
- Parameters
- Parameter types
- Parameter descriptions

---

### Q42: When should a tool return a string vs an object?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> Tools should always return **strings** (or convertible to string).

**Why**: LLM processes text. Objects don't work.

**Wrong**:
```python
def get_data() -> dict:
    return {"name": "John", "age": 30}  # ✗ LLM can't parse dict
```

**Right**:
```python
def get_data() -> str:
    data = {"name": "John", "age": 30}
    return json.dumps(data)  # ✓ LLM sees JSON string
```

---

### Q43: What is the purpose of the source field in TextMessage?
**Expected Answer Length**: 1 minute

**Sample Answer**:
> `source` identifies which agent sent the message.

```python
TextMessage(
    content="Analysis complete",
    source="analyzer"  # Identifies sender
)
```

**Why**: When multiple agents speak, you need to know who said what.

---

### Q44: What does TaskResult.stop_reason tell you?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> `stop_reason` explains why execution stopped.

**Possible values**:
- `"response"` - Agent responded (normal)
- `"max_tool_iterations"` - Tool limit reached
- `"max_turns"` - Turn limit reached
- `"cancelled"` - User cancelled
- `"error"` - Exception occurred

**Usage**:
```python
result = await team.run(...)
if result.stop_reason == "max_turns":
    # Incomplete, try again
elif result.stop_reason == "response":
    # Normal completion
```

---

### Q45: How does the LLM know which tools are available?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> Tools are converted to JSON schema and sent as part of the LLM prompt.

**Process**:
```python
def web_search(query: str) -> str:
    """Search the web."""
    ...

# Converted to schema:
{
    "name": "web_search",
    "description": "Search the web.",
    "parameters": {
        "type": "object",
        "properties": {
            "query": {"type": "string"}
        }
    }
}

# Sent to LLM as: "Available tools: [schema1, schema2, ...]"
```

---

## Additional Level 2: Implementation Extended (15 more questions)

### Q46: What happens if a tool raises an exception?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> Exception is caught and returned as error in ToolCallSummaryMessage.

**Example**:
```python
def divide(a: int, b: int) -> str:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return str(a / b)

# Agent calls: divide(10, 0)
# ToolCallSummaryMessage contains: "ValueError: Cannot divide by zero"
# Agent sees error and can retry or fallback
```

---

### Q47: How do you handle tool timeouts?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
import asyncio

def web_search_with_timeout(query: str) -> str:
    try:
        # Timeout prevents hanging
        result = asyncio.run(
            asyncio.wait_for(
                api.search(query),
                timeout=5.0  # 5 second timeout
            )
        )
        return result
    except asyncio.TimeoutError:
        return "Search timed out. Please try again."
    except Exception as e:
        return f"Error: {e}"
```

**Alternative**: Configure timeout at agent level (not standard in AutoGen, but use task-level timeouts).

---

### Q48: What is the difference between passing tools as list vs using @tool decorator?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:

**List approach**:
```python
def tool1(x: int) -> str: return str(x)
def tool2(y: str) -> str: return y.upper()

agent = AssistantAgent(tools=[tool1, tool2])
```

**Decorator approach**:
```python
from autogen_ext.models.openai import Tool

@tool
def tool1(x: int) -> str: 
    """Description."""
    return str(x)

agent = AssistantAgent(tools=[tool1])
```

**Difference**: Both work; decorator is cleaner, list is more explicit.

---

### Q49: How do you provide parameter descriptions for tools?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> Use `Annotated` type hints with descriptions.

```python
from typing import Annotated

def search(
    query: Annotated[str, "The search query string"],
    max_results: Annotated[int, "Maximum number of results to return"] = 10
) -> str:
    """Search the web."""
    ...
```

**LLM sees**:
- Parameter name: query
- Type: string
- Description: "The search query string"

---

### Q50: What is the difference between await agent.run() and async for event in agent.run_stream()?
**Expected Answer Length**: 2 minutes

**Sample Answer**:

```python
# run() - blocking, wait for result
result = await agent.run(task="...")
# Returns: TaskResult (final state only)
print(result.messages[-1])

# run_stream() - streaming, see events
async for event in agent.run_stream(task="..."):
    # Get events as they happen
    if event["type"] == "message_created":
        print(f"New message: {event['message'].content}")
```

**When to use**:
- **run()**: Final result needed
- **run_stream()**: Real-time visibility, monitoring, debugging

---

### Q51: How do you debug if an agent keeps calling the same tool?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
# Enable debugging with streaming
agent_messages = []
async for event in agent.run_stream(task="..."):
    if event["type"] == "message_created":
        msg = event["message"]
        agent_messages.append(msg)
        print(f"{msg.source}: {msg.content[:100]}...")

# Check for repeated tool calls
tool_calls = [m for m in agent_messages if "tool" in str(m)]
if len(tool_calls) > 10:
    print("WARNING: Many tool calls - possible loop")
```

**Solutions**:
- Reduce max_tool_iterations
- Add better termination logic
- Improve system message

---

### Q52: What is the best way to structure a tool that needs multiple steps?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> Option 1: Multiple tools (recommended)
```python
def fetch_data(id: str) -> str: return db.get(id)
def process_data(data: str) -> str: return process(data)
def save_result(result: str) -> str: return db.save(result)

# Agent orchestrates: fetch → process → save
```

> Option 2: Single tool that does all
```python
def fetch_process_save(id: str) -> str:
    data = db.get(id)
    result = process(data)
    db.save(result)
    return result
```

**Recommendation**: Multiple small tools (more flexibility, clearer to agent).

---

### Q53: How do you pass configuration to tools?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> Use closures or classes.

**Closure approach**:
```python
def create_search_tool(api_key):
    def search(query: str) -> str:
        api = SearchAPI(api_key)
        return api.search(query)
    return search

tool = create_search_tool(os.getenv("API_KEY"))
agent = AssistantAgent(tools=[tool])
```

**Class approach**:
```python
class SearchTool:
    def __init__(self, api_key):
        self.api = SearchAPI(api_key)
    
    def __call__(self, query: str) -> str:
        return self.api.search(query)

tool = SearchTool(os.getenv("API_KEY"))
```

---

### Q54: What's the best way to handle authentication in tools?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
# WRONG - Hardcoded credentials
def get_data() -> str:
    api = API(key="sk-secret-123")  # ✗ Exposed!

# RIGHT - Environment variables
def get_data() -> str:
    api_key = os.getenv("MY_API_KEY")
    if not api_key:
        return "Error: API key not configured"
    api = API(key=api_key)
    return api.get_data()

# Even better - Initialize once
api = API(key=os.getenv("MY_API_KEY"))

def get_data() -> str:
    return api.get_data()  # Already authenticated
```

---

### Q55: How do you unit test tools in isolation?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
import unittest
from unittest.mock import patch

class TestTools(unittest.TestCase):
    @patch('requests.get')
    def test_web_search(self, mock_get):
        mock_get.return_value.json.return_value = {"results": [...]}
        
        result = web_search("Python")
        
        self.assertIn("Python", result)
        mock_get.assert_called_once()

    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            divide(10, 0)
```

**Best practice**: Mock external dependencies, test error cases.

---

### Q56: What's the purpose of the tools parameter on CustomAgent?
**Expected Answer Length**: 1 minute

**Sample Answer**:
> Tools parameter makes functions available to the agent.

```python
class MyAgent(BaseChatAgent):
    def __init__(self, name, tools):
        self.name = name
        self.tools = tools  # Available during execution
        # Agent can access via self.tools[name]
```

**Tools are not automatic** - agent must explicitly use them (LLM decides).

---

### Q57: How do you make a tool that returns structured data?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
```python
import json

def get_user_data(user_id: str) -> str:
    """Get user information as JSON."""
    user = db.query(user_id)
    # Return as JSON string
    return json.dumps({
        "id": user.id,
        "name": user.name,
        "email": user.email
    })

# Agent receives JSON string and can parse it
```

**Key**: Return JSON string, not dict. Agent (LLM) receives and processes text.

---

### Q58: How do you handle long-running tools?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
```python
import asyncio

async def long_running_tool(task_id: str) -> str:
    """Check status of long task."""
    # For long tasks, return status instead of waiting
    status = db.get_task_status(task_id)
    if status == "in_progress":
        return "Task in progress. Check again later."
    elif status == "completed":
        return "Task completed with result: ..."
    else:
        return "Task failed."
```

**Pattern**: Don't block agent. Return status/result when ready.

---

### Q59: What happens if a tool modifies state?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
> Tool is responsible for state changes. Agent doesn't manage it.

**Example**:
```python
def save_document(title: str, content: str) -> str:
    doc = Document(title=title, content=content)
    db.save(doc)  # Modifies state
    return f"Saved: {title}"

# Agent calls tool
# Side effect: Document saved to DB
# Agent gets confirmation
```

**Idempotency**: If tool might be called twice, ensure it's idempotent.

---

### Q60: How do you version tools as the system evolves?
**Expected Answer Length**: 1.5 minutes

**Sample Answer**:
```python
# Version 1
def search_v1(query: str) -> str:
    """Search the web."""
    return results

# Version 2 - backwards compatible
def search_v2(
    query: str,
    domain: str = None  # New parameter, optional
) -> str:
    """Search the web with optional domain filter."""
    if domain:
        query = f"{query} site:{domain}"
    return results

# Transition: replace v1 with v2 gradually
agent = AssistantAgent(tools=[search_v2])
```

---

## Additional Level 3: System Design Extended (10 more questions)

### Q61: How would you implement a multi-stage approval workflow?
**Expected Answer Length**: 3 minutes

**Sample Answer**:
```
Stage 1: Generation
└─ Agent generates proposal

Stage 2: First Approval
├─ Manager 1 reviews
├─ Approves → continues
└─ Rejects → back to generation

Stage 3: Second Approval
├─ Manager 2 reviews
├─ Approves → continues
└─ Rejects → back to stage 1

Stage 4: Execution
└─ Execute approved proposal
```

**Implementation**:
```python
generation_agent = AssistantAgent(...)
manager_1 = UserProxyAgent(name="manager1", input_func=...)
manager_2 = UserProxyAgent(name="manager2", input_func=...)
executor = AssistantAgent(...)

team = RoundRobinGroupChat(
    participants=[generation_agent, manager_1, manager_2, executor],
    max_turns=15,
    termination_condition=StopMessageTermination(stop_words=["EXECUTED"])
)
```

---

### Q62: How would you build a recommendation system with AutoGen?
**Expected Answer Length**: 3 minutes

**Sample Answer**:
```
User Query
    ↓
Collector Agent
├─ Gathers user preferences
├─ Fetches available options
├─ Calls filtering tool
    ↓
Ranker Agent
├─ Scores each option
├─ Weighs preferences
├─ Calls ranking tool
    ↓
Explainer Agent
├─ Explains top-3 recommendations
├─ Provides rationale
├─ Calls explanation tool
    ↓
Final Recommendations
```

**Tools**:
- `filter_options(criteria)` - returns matching items
- `score_item(item, preferences)` - returns score
- `explain_recommendation(item)` - returns explanation

---

### Q63: How would you implement retry logic with exponential backoff?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
import asyncio

async def call_with_retry(agent, task, max_retries=3):
    for attempt in range(max_retries):
        try:
            result = await agent.run(task=task)
            if result.stop_reason != "error":
                return result
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            
            wait_time = 2 ** attempt  # 1s, 2s, 4s
            await asyncio.sleep(wait_time)
    
    return None

# Usage
result = await call_with_retry(agent, task)
```

---

### Q64: How would you design a system that handles both sync and async operations?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
def sync_tool(query: str) -> str:
    """Synchronous tool."""
    return results  # Immediate

async def async_wrapper(query: str) -> str:
    """Wrap async as sync."""
    import asyncio
    result = await slow_api.query(query)
    return result

# Agent uses wrapper
agent = AssistantAgent(tools=[sync_tool, async_wrapper])
```

**Key**: Keep tools synchronous. Use wrappers for async operations.

---

### Q65: How would you implement caching to reduce API calls?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
from functools import lru_cache

@lru_cache(maxsize=100)
def cached_search(query: str) -> str:
    """Cache search results."""
    return api.search(query)

# Or custom cache
cache = {}

def search_with_cache(query: str) -> str:
    if query in cache:
        return cache[query]
    
    result = api.search(query)
    cache[query] = result
    return result

agent = AssistantAgent(tools=[search_with_cache])
```

---

### Q66: How would you implement cost controls in a production system?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
class CostController:
    def __init__(self, budget_per_hour=100):
        self.budget = budget_per_hour
        self.spent = 0
    
    async def run_with_budget(self, agent, task):
        result = await agent.run(task=task)
        
        tokens = result.metadata.get('tokens', 0)
        cost = (tokens / 1000) * 0.05  # Simplified
        
        self.spent += cost
        
        if self.spent > self.budget:
            raise BudgetExceeded(f"Spent ${self.spent}")
        
        return result

controller = CostController(budget_per_hour=50)
result = await controller.run_with_budget(agent, task)
```

---

### Q67: How would you implement observability for debugging?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
import logging
import json

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

async def run_with_observability(team, task):
    logger.info(f"Starting task: {task}")
    
    event_log = []
    async for event in team.run_stream(task=task):
        event_log.append(event)
        
        if event["type"] == "message_created":
            logger.info(f"Message: {event['message'].source}")
        
        elif event["type"] == "tool_call_starting":
            logger.info(f"Tool: {event['tool']}")
    
    # Save for debugging
    with open("execution_log.json", "w") as f:
        json.dump(event_log, f)
    
    return event_log
```

---

### Q68: How would you handle model failures with fallback?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
async def create_agent_with_fallback(task):
    models = [
        OpenAIChatCompletionClient(model="gpt-4"),
        OpenAIChatCompletionClient(model="gpt-4-turbo"),
        OpenAIChatCompletionClient(model="gpt-3.5-turbo"),
    ]
    
    for i, model in enumerate(models):
        try:
            agent = AssistantAgent(model_client=model)
            result = await agent.run(task=task)
            logger.info(f"Success with model {i}")
            return result
        except Exception as e:
            logger.warning(f"Model {i} failed: {e}")
            if i == len(models) - 1:
                raise

# Usage
result = await create_agent_with_fallback(task)
```

---

### Q69: How would you implement A/B testing for agents?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
import random

async def ab_test_agents(task, agent_a, agent_b):
    # 50/50 split
    if random.random() < 0.5:
        selected = agent_a
        variant = "A"
    else:
        selected = agent_b
        variant = "B"
    
    result = await selected.run(task=task)
    
    # Log for analysis
    log_entry = {
        "variant": variant,
        "task": task,
        "success": result.stop_reason == "response",
        "tokens": result.metadata.get('tokens'),
        "latency": result.metadata.get('duration')
    }
    
    db.save_test_result(log_entry)
    return result, variant

result, variant = await ab_test_agents(task, agent_a, agent_b)
```

---

### Q70: How would you implement state persistence for resumable workflows?
**Expected Answer Length**: 2 minutes

**Sample Answer**:
```python
async def resumable_workflow(task_id):
    # Try to load previous state
    previous_state = db.get_checkpoint(task_id)
    
    if previous_state:
        team.load_state(previous_state)
        logger.info(f"Resuming from checkpoint")
        # Continue where it left off
        result = await team.run(task="Continue processing")
    else:
        logger.info(f"Starting new task")
        result = await team.run(task=task_id)
    
    # Save checkpoint for potential resumption
    db.save_checkpoint(task_id, team.save_state())
    
    return result
```

---

## Quick Reference Cheat Sheet

**Single Agent Loop**:
1. Receive task
2. Build context (history + tools + system)
3. Call LLM
4. If tool → execute → loop to step 2
5. If response → return

**Team Execution**:
1. User task
2. Agent 1 speaks
3. Agent 2 receives all history
4. Agent 2 speaks
5. Repeat until max_turns or termination

**Error Recovery**:
- Transient errors → retry with backoff
- Permanent errors → fallback
- No recovery → escalate

**Cost Control**:
- Monitor tokens per request
- Set budget limits
- Use cheaper models for simple tasks
- Cache results

**Debugging**:
- Use run_stream() for visibility
- Log messages and tool calls
- Check stop_reason
- Monitor token usage

---

## Red Flags: What Interviewers Watch For

### Conceptual Misunderstandings
- ❌ "Agents are just LLMs with tools" (Missing orchestration layer)
- ❌ "Tools execute code directly" (Tools return strings)
- ❌ "Handoff is automatic" (Agent explicitly sends HandoffMessage)
- ❌ "max_tool_iterations limits team turns" (Limits individual agent iterations)
- ❌ "RoundRobin and Selector choose same next agent" (Selector is AI-driven)

### Implementation Mistakes
- ❌ Hardcoding API keys in tools
- ❌ Not handling tool exceptions
- ❌ Assuming tool always succeeds
- ❌ Not setting max_turns and termination_condition
- ❌ Forgetting state persistence

### Production Oversights
- ❌ No monitoring or logging
- ❌ No cost tracking
- ❌ No retry logic
- ❌ No fallback strategies
- ❌ No error classification

---

## Interview Tips - Advanced

### How to Answer "Design X System"
1. **Start simple**: Single agent → multiple agents
2. **Add constraints**: Budget, latency, reliability
3. **Show tradeoffs**: Why RoundRobin vs Selector?
4. **Include observability**: How do you know it works?
5. **Handle failures**: What if X fails?

### Common Follow-ups
**Q: "How would you scale this?"**
- Separate agents on different machines
- Use message queues
- Implement load balancing

**Q: "How would you reduce costs?"**
- Use cheaper models for simple tasks
- Cache tool results
- Reduce tool iterations

**Q: "How would you ensure reliability?"**
- Retry with backoff
- Fallback strategies
- Health checks
- Circuit breakers

---

## References

- [AutoGen Documentation](https://microsoft.github.io/autogen/)
- [Completed Documentation Files](./README.md)
- [API Reference](29-api-reference.md)
- [Troubleshooting Guide](28-troubleshooting.md)
- [autogen-production-strategies.md](autogen-production-strategies.md)
- [autogen-evaluation-and-interview.md](autogen-evaluation-and-interview.md)

### Before the Interview
- ✅ Read all 7 completed documents thoroughly
- ✅ Understand distinction between LLM, Agent, Tool
- ✅ Know the three layers of architecture
- ✅ Practice explaining single-agent loop
- ✅ Review parameter tables in 29-api-reference.md

### During the Interview
- ✅ Start with big picture, then drill down
- ✅ Use concrete examples from real code
- ✅ Explain the "why" not just the "what"
- ✅ Be honest about what you don't know
- ✅ Show understanding of trade-offs

### Common Interview Patterns
1. **"Explain X"** → Give 1-minute overview, then expand
2. **"How would you..."** → Start with architecture, then implementation
3. **"What's the difference..."** → Use comparison table mentally
4. **"Design a system..."** → Start simple, add complexity

### Red Flags to Avoid
- ❌ Confusing Agent with LLM
- ❌ Forgetting about max_tool_iterations
- ❌ Not knowing purpose of model_client abstraction
- ❌ Thinking tools execute code directly
- ❌ Assuming handoffs are automatic routing

---

## References

- [AutoGen Documentation](https://microsoft.github.io/autogen/)
- [Completed Documentation Files](./README.md)
- [API Reference](29-api-reference.md)
- [Troubleshooting Guide](28-troubleshooting.md)
