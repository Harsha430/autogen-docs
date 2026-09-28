# Part 2: AutoGen Architecture

## Overall Architecture

AutoGen is layered architecture with three primary levels:

```
┌────────────────────────────────────────────┐
│         Your Application Code              │
│                                            │
│  from autogen_agentchat.agents import     │
│      AssistantAgent, UserProxyAgent       │
│  from autogen_agentchat.teams import      │
│      RoundRobinGroupChat                  │
└────────────────┬─────────────────────────┘
                 │
        ┌────────▼─────────┐
        │  AgentChat       │
        │  (High-level)    │
        │                  │
        │ • Agents         │
        │ • Teams          │
        │ • Messages       │
        │ • Tools          │
        │ • run()          │
        │ • run_stream()   │
        └────────┬─────────┘
                 │
        ┌────────▼──────────┐
        │  AutoGen Core    │
        │  (Low-level)     │
        │                  │
        │ • AgentRuntime   │
        │ • Events         │
        │ • Routing        │
        │ • RPC            │
        │ • Subscriptions  │
        └────────┬──────────┘
                 │
        ┌────────▼──────────────┐
        │  Model Clients        │
        │                       │
        │ • OpenAIChatClient    │
        │ • AzureOpenAIChatClient
        │ • Custom clients      │
        └────────┬──────────────┘
                 │
        ┌────────▼──────────────┐
        │  External LLMs        │
        │                       │
        │ • OpenAI API          │
        │ • Azure OpenAI        │
        │ • Other providers     │
        └───────────────────────┘
```

## Why This Architecture?

**Separation of concerns**:
- **AgentChat**: High-level agent orchestration (what you use most)
- **Core**: Event-driven runtime (when you need low-level control)
- **Model Clients**: LLM provider abstraction (swap providers easily)

This allows:
- Simple applications use only AgentChat
- Complex applications can drop to Core for advanced patterns
- Model providers are interchangeable

---

## Level 1: AgentChat (High-Level API)

**Purpose**: User-friendly API for building multi-agent systems.

**You use this 95% of the time.**

### Core Components

#### Agents
```python
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent

# AI-powered agent
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[web_search]
)

# Human-side endpoint
human = UserProxyAgent(name="human")
```

#### Teams
```python
from autogen_agentchat.teams import RoundRobinGroupChat

team = RoundRobinGroupChat(
    participants=[researcher, analyzer, writer],
    max_turns=10
)
```

#### Messages
```python
from autogen_agentchat.messages import TextMessage, HandoffMessage

# Agents exchange messages
msg = TextMessage(
    content="Here's my analysis...",
    source="analyzer"
)
```

#### Execution
```python
# Run complete task
result = await team.run(
    task="Research the latest AI developments"
)

# Or stream events for observability
async for event in team.run_stream(
    task="Research the latest AI developments"
):
    print(event)
```

### AgentChat Architecture Diagram

```
┌──────────────────────────────────────────┐
│  AgentChat Layer                         │
├──────────────────────────────────────────┤
│                                          │
│  ┌──────────────────────────────────┐  │
│  │ Agent Implementations             │  │
│  │ • AssistantAgent (AI-powered)    │  │
│  │ • UserProxyAgent (Human input)   │  │
│  │ • BaseChatAgent (Custom)         │  │
│  └──────────┬───────────────────────┘  │
│             │                           │
│  ┌──────────▼───────────────────────┐  │
│  │ Team Orchestration                │  │
│  │ • RoundRobinGroupChat            │  │
│  │ • SelectorGroupChat              │  │
│  │ • Swarm                          │  │
│  └──────────┬───────────────────────┘  │
│             │                           │
│  ┌──────────▼───────────────────────┐  │
│  │ Message & Tool System             │  │
│  │ • TextMessage                     │  │
│  │ • HandoffMessage                  │  │
│  │ • Tool schema validation          │  │
│  │ • Tool execution                  │  │
│  └──────────┬───────────────────────┘  │
│             │                           │
│  ┌──────────▼───────────────────────┐  │
│  │ Execution Control                 │  │
│  │ • run() - single execution        │  │
│  │ • run_stream() - event streaming  │  │
│  │ • Termination conditions          │  │
│  │ • Cancellation tokens             │  │
│  └──────────────────────────────────┘  │
│                                          │
└────────────────┬─────────────────────────┘
                 │
      ┌──────────▼──────────┐
      │  AutoGen Core       │
      │  (underneath)       │
      └─────────────────────┘
```

---

## Level 2: AutoGen Core (Low-Level Runtime)

**Purpose**: Event-driven runtime for advanced multi-agent patterns.

**You use this when**:
- AgentChat doesn't support your use case
- You need direct RPC between agents
- You're building a custom agent framework
- You need fine-grained execution control

### Core Components

#### AgentRuntime
The execution engine that manages agent lifecycle and message routing.

```python
from autogen_core import AgentRuntime

runtime = AgentRuntime()

# Add agents to runtime
await runtime.add_agent(
    agent_id="researcher",
    agent_instance=researcher_agent
)
```

#### Agent Identification
```python
from autogen_core import AgentId, AgentType

# Each agent has an ID
agent_id = AgentId(
    name="researcher",
    type=AgentType.AGENT
)
```

#### Topics and Subscriptions
```python
from autogen_core import TopicId, Subscription

# Agents publish to topics
topic = TopicId(name="research_results")

# Agents subscribe to topics
subscription = Subscription(
    topic=topic,
    agent_id=agent_id
)

# When anything publishes to research_results,
# the subscribed agent receives it
```

#### Message Handlers
```python
from autogen_core import message_handler

@message_handler
async def on_message(
    self,
    message: str,
    ctx: MessageHandlerContext
) -> str:
    """Handle incoming message"""
    return f"Processed: {message}"
```

#### Events
```python
from autogen_core import event

@event
async def on_task_complete(
    self,
    result: TaskResult,
    ctx: MessageHandlerContext
) -> None:
    """Handle task completion event"""
    print(f"Task complete: {result}")
```

#### RPC (Remote Procedure Calls)
```python
from autogen_core import MessageContext

# Direct agent-to-agent communication
response = await ctx.send_message(
    message="Analyze this data",
    recipient=analyzer_agent_id
)
```

### Core vs AgentChat Comparison

| Aspect | AgentChat | Core |
|--------|-----------|------|
| **Learning curve** | Easy | Steep |
| **Use case** | Most applications | Advanced/custom |
| **Message passing** | Implicit via teams | Explicit subscriptions |
| **Routing** | Handled by team type | Manual routing |
| **Event handling** | Limited | Full event system |
| **Debugging** | Straightforward | Complex but explicit |
| **Performance** | Good | Better for complex routing |

### Core Architecture Diagram

```
┌──────────────────────────────────────┐
│  AutoGen Core Layer                  │
├──────────────────────────────────────┤
│                                      │
│  ┌──────────────────────────────┐   │
│  │ AgentRuntime                 │   │
│  │ • Lifecycle management       │   │
│  │ • Message routing            │   │
│  │ • Agent registration         │   │
│  └────────────┬─────────────────┘   │
│               │                     │
│  ┌────────────▼──────────────────┐  │
│  │ Pub/Sub System                │  │
│  │ • Topics                      │  │
│  │ • Subscriptions               │  │
│  │ • Broadcast messaging         │  │
│  └────────────┬──────────────────┘  │
│               │                     │
│  ┌────────────▼──────────────────┐  │
│  │ RPC System                    │  │
│  │ • Direct agent calls          │  │
│  │ • Request/response            │  │
│  │ • Context propagation         │  │
│  └────────────┬──────────────────┘  │
│               │                     │
│  ┌────────────▼──────────────────┐  │
│  │ Event System                  │  │
│  │ • @message_handler            │  │
│  │ • @event decorator            │  │
│  │ • MessageContext & Context    │  │
│  └──────────────────────────────┘  │
│                                      │
└──────────────────────────────────────┘
```

---

## Level 3: Model Clients

**Purpose**: Abstraction layer for different LLM providers.

This separates agent logic from provider-specific API details.

### Problem Without Model Clients

```python
# WITHOUT model_client abstraction
class MyAgent:
    def __init__(self):
        self.openai_key = os.getenv("OPENAI_KEY")
        self.client = OpenAI(api_key=self.openai_key)
    
    async def process(self, task):
        response = await self.client.chat.completions.create(
            model="gpt-4",
            messages=[...],
            temperature=0.7,
            max_tokens=2000,
            # OpenAI-specific parameters
        )
        return response.choices[0].message.content

# Problem: Agent is tightly coupled to OpenAI API
# Switching providers requires rewriting agent
```

### Solution With Model Clients

```python
# WITH model_client abstraction
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY")
)

agent = AssistantAgent(
    name="researcher",
    model_client=model_client
)

# To switch providers, just swap model_client:
model_client = AzureOpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("AZURE_KEY"),
    endpoint=os.getenv("AZURE_ENDPOINT")
)

# Same agent code works with Azure
agent = AssistantAgent(
    name="researcher",
    model_client=model_client
)
```

### Model Client Architecture

```
┌──────────────────────────────────┐
│  AssistantAgent                  │
│  • Knows what to do             │
│  • Calls model_client           │
└──────────────┬───────────────────┘
               │
      ┌────────▼──────────────────┐
      │  model_client            │
      │  (abstraction)           │
      │                          │
      │  interface:              │
      │  • create_chat_response()│
      │  • get_usage()           │
      │  • stream()              │
      └────────┬─────────────────┘
               │
      ┌────────▼─────────────────────┐
      │  Concrete Implementation     │
      │                             │
      │  OpenAIChatClient:          │
      │  └─ OpenAI API calls       │
      │                             │
      │  AzureOpenAIChatClient:     │
      │  └─ Azure OpenAI API calls  │
      │                             │
      │  CustomChatClient:          │
      │  └─ Your LLM provider       │
      └─────────────────────────────┘
```

---

## Component Relationships

### How Components Interact

```
┌─────────────────────────────────────────────────────┐
│  User Code                                          │
│                                                     │
│  1. Create model_client                            │
│     ↓                                              │
│  2. Create agents (with model_client)              │
│     ↓                                              │
│  3. Create team (with agents)                      │
│     ↓                                              │
│  4. Call team.run() or team.run_stream()           │
│     ↓                                              │
└──────────────┬──────────────────────────────────────┘
               │
    ┌──────────▼────────────┐
    │  AgentChat Layer      │
    │                       │
    │  • Team receives      │
    │    user task          │
    │                       │
    │  • Selects which      │
    │    agent speaks       │
    │                       │
    │  • Agent receives     │
    │    task               │
    └──────────┬────────────┘
               │
    ┌──────────▼──────────────┐
    │  Agent Logic            │
    │                         │
    │  • Should I call tool?  │
    │    ├─ YES: Call tool    │
    │    └─ NO: Respond       │
    │                         │
    │  • Feed to LLM via      │
    │    model_client         │
    └──────────┬──────────────┘
               │
    ┌──────────▼──────────────┐
    │  Model Client           │
    │                         │
    │  • Format message       │
    │  • Call API             │
    │  • Handle response      │
    └──────────┬──────────────┘
               │
    ┌──────────▼──────────────┐
    │  LLM Provider           │
    │                         │
    │  • Process request      │
    │  • Generate response    │
    │  • Return result        │
    └──────────┬──────────────┘
               │
    ┌──────────▼──────────────┐
    │  Back to Agent          │
    │                         │
    │  • Parse response       │
    │  • Extract tool calls   │
    │  • Or extract text      │
    │  • Next iteration?      │
    └─────────────────────────┘
```

---

## Data Flow Through Architecture

### Simple Single-Agent Example

```
User: "Search for Python news"
        ↓
        ├─────────────────────────────────┐
        │ AgentChat (High-level)          │
        │                                 │
        │ AssistantAgent.on_messages()    │
        │ received = [TextMessage(...)]   │
        └──────────────┬──────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ Agent Decision Logic            │
        │                                │
        │ 1. Should call web_search?     │
        │    → Yes (LLM decides)         │
        │                                │
        │ 2. Prepare LLM request:        │
        │    - conversation history     │
        │    - available tools          │
        │    - system message           │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ Model Client (Abstraction)      │
        │                                │
        │ OpenAIChatClient.create(...)   │
        │ - Formats messages             │
        │ - Includes tools as JSON       │
        │ - Sets temperature, tokens     │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ OpenAI API                      │
        │                                │
        │ LLM processes:                 │
        │ - Sees search query            │
        │ - Sees web_search tool         │
        │ - Generates tool_call JSON     │
        │ - Returns: {                   │
        │    "tool_calls": [{            │
        │      "name": "web_search",     │
        │      "args": {...}             │
        │    }]                          │
        │  }                             │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ Back to Agent                   │
        │                                │
        │ 1. Parse tool_call            │
        │ 2. Execute web_search()       │
        │ 3. Get results                │
        │                                │
        │ 4. Add to conversation:        │
        │    - Tool call request         │
        │    - Tool execution event     │
        │    - Tool result              │
        │                                │
        │ 5. Call LLM again with       │
        │    tool result                │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ OpenAI API (Second Call)        │
        │                                │
        │ LLM sees tool result           │
        │ Generates final response       │
        │ Returns text (no tool calls)   │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ Final Result                    │
        │                                │
        │ Agent returns to caller:       │
        │ "Python news: [results]"       │
        └─────────────────────────────────┘
```

---

## When to Use Each Layer

### Use AgentChat When:
✅ Building typical multi-agent systems  
✅ Using predefined team patterns  
✅ Need simple message passing  
✅ Want high-level abstractions  
✅ Learning AutoGen  
✅ Most production applications  

### Drop to Core When:
❌ AgentChat doesn't support your pattern  
⚡ You need RPC between agents  
📡 You need pub/sub communication  
🎛️ Fine-grained execution control  
🔧 Building custom agent frameworks  
🏗️ Complex routing logic  

### Why Not Drop to Core?
Core is more powerful but also:
- More complex
- Requires understanding event systems
- More code to write
- Harder to debug

**Recommendation**: Start with AgentChat, drop to Core only if needed.

---

## Extensions and Integrations

AutoGen can be extended:

### Memory Systems
```
Agent ──────────────────┐
       (what it knows)  │
                        ├─→ Memory System
       (can retrieve)   │
       ─────────────────┘
```

### Observability
```
Agent ──────────────────┐
       (emits events)   │
                        ├─→ Logging/Tracing/Metrics
       (can be tracked) │
       ─────────────────┘
```

### Custom Tools
```
Agent ──────────────────┐
       (calls tool)     │
                        ├─→ Tool System
       (gets result)    │
       ─────────────────┘
```

---

## Architecture Design Philosophy

### 1. Layering
Each layer has clear responsibility and doesn't leak concerns upward.

### 2. Abstraction
Model clients abstract provider details from agents.

### 3. Composition
Complex systems built by composing simple components.

### 4. Message-Driven
Communication through messages, not shared state.

### 5. Extensibility
Custom agents, tools, and model clients are first-class citizens.

---

## Next Steps

Now that you understand the overall architecture:

- Move to **[03-model-client.md](03-model-client.md)** to understand LLM provider abstraction
- Or jump to **[04-assistant-agent.md](04-assistant-agent.md)** for agent deep-dive

---

## References

- [AutoGen Architecture Documentation](https://microsoft.github.io/autogen/docs/topics/)
- [AutoGen GitHub - Architecture](https://github.com/microsoft/autogen/docs/architecture)

---

## Version Notes

- **Current Version**: AutoGen 0.2.x
- **AgentChat**: Stable API for most use cases
- **Core**: Event-driven runtime (less common but powerful)
