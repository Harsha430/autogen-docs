# Part 29: Complete API Reference

## Quick Index

- [AssistantAgent](#assistantagent)
- [OpenAIChatCompletionClient](#openaiatchatcompletionclient)
- [RoundRobinGroupChat](#roundrobingroupchat)
- [TextMessage](#textmessage)
- [run() / run_stream()](#run--run_stream)

---

## AssistantAgent

**Import**:
```python
from autogen_agentchat.agents import AssistantAgent
```

### Constructor Parameters

| Parameter | Type | Required | Default | Purpose |
|-----------|------|----------|---------|---------|
| `name` | `str` | **Yes** | - | Agent identifier in team |
| `model_client` | `ChatCompletionClient` | **Yes** | - | LLM provider (OpenAI, Azure, etc) |
| `system_message` | `str` | No | `"You are a helpful assistant."` | Agent role and instructions for LLM |
| `tools` | `list` | No | `None` | Python functions agent can call |
| `model_context` | `ModelContext` | No | `None` | Context window management |
| `memory` | `Memory` | No | `None` | Long-term memory system |
| `handoffs` | `list[str]` | No | `None` | Agent names this agent can transfer to |
| `max_tool_iterations` | `int` | No | `10` | Max tool calls before stopping |
| `reflect_on_tool_use` | `bool` | No | `False` | Add verification step after tools |
| `workbench` | `WorkbenchType` | No | `None` | Tool execution sandbox (experimental) |

### Methods

#### on_messages()
**Signature**: `async def on_messages(self, messages: list[TextMessage]) -> TextMessage`

**Purpose**: Process incoming messages

**Parameters**:
- `messages`: List of TextMessage objects

**Returns**: TextMessage response

**Usage**:
```python
# Usually called automatically by team
response = await agent.on_messages([incoming_message])
```

#### run()
**Signature**: `async def run(self, task: str) -> TaskResult`

**Purpose**: Execute a single task to completion

**Parameters**:
- `task`: Task description string

**Returns**: TaskResult with messages and metadata

**Usage**:
```python
result = await agent.run(task="Research Python 3.13")
print(result.messages[-1].content)  # Final response
```

#### run_stream()
**Signature**: `async def run_stream(self, task: str) -> AsyncIterator`

**Purpose**: Execute task with event streaming

**Parameters**:
- `task`: Task description string

**Returns**: Async generator yielding events

**Usage**:
```python
async for event in agent.run_stream(task="..."):
    print(event)
```

---

## OpenAIChatCompletionClient

**Import**:
```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
```

### Constructor Parameters

| Parameter | Type | Required | Default | Purpose |
|-----------|------|----------|---------|---------|
| `model` | `str` | **Yes** | - | Model name (gpt-4, gpt-3.5-turbo) |
| `api_key` | `str` | **Yes** | - | OpenAI API key |
| `base_url` | `str` | No | OpenAI default | API endpoint (for proxies) |
| `api_version` | `str` | No | - | API version |
| `timeout` | `int` | No | 300 | Request timeout in seconds |
| `temperature` | `float` | No | 1.0 | Response randomness (0-2) |
| `max_tokens` | `int` | No | None | Max output tokens |
| `top_p` | `float` | No | 1.0 | Nucleus sampling |
| `seed` | `int` | No | None | For reproducible outputs |
| `extra_create_args` | `dict` | No | {} | Additional OpenAI parameters |

### Methods

#### create()
**Signature**: `async def create(self, messages: list, tools: list = None, **kwargs) -> ChatCompletion`

**Purpose**: Call OpenAI API

**Parameters**:
- `messages`: Chat messages
- `tools`: Available tools
- Additional OpenAI parameters

**Returns**: ChatCompletion response

---

## AzureOpenAIChatCompletionClient

**Import**:
```python
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
```

### Constructor Parameters

| Parameter | Type | Required | Default | Purpose |
|-----------|------|----------|---------|---------|
| `model` | `str` | **Yes** | - | Azure deployment name |
| `api_key` | `str` | **Yes** | - | Azure OpenAI API key |
| `api_version` | `str` | **Yes** | - | Azure API version |
| `azure_endpoint` | `str` | **Yes** | - | Azure resource endpoint |
| `timeout` | `int` | No | 300 | Request timeout in seconds |
| `temperature` | `float` | No | 1.0 | Response randomness |
| `max_tokens` | `int` | No | None | Max output tokens |

---

## RoundRobinGroupChat

**Import**:
```python
from autogen_agentchat.teams import RoundRobinGroupChat
```

### Constructor Parameters

| Parameter | Type | Required | Default | Purpose |
|-----------|------|----------|---------|---------|
| `participants` | `list[Agent]` | **Yes** | - | Agents in fixed order |
| `max_turns` | `int` | No | None | Max conversation turns |
| `termination_condition` | `TerminationCondition` | No | None | Stop condition |
| `model_client` | `ChatCompletionClient` | No | None | For group decisions (if needed) |

### Methods

#### run()
**Signature**: `async def run(self, task: str) -> TaskResult`

**Purpose**: Run team to completion

**Returns**: TaskResult with all messages

#### run_stream()
**Signature**: `async def run_stream(self, task: str) -> AsyncIterator`

**Purpose**: Stream team execution

**Returns**: Async generator of events

---

## TextMessage

**Import**:
```python
from autogen_agentchat.messages import TextMessage
```

### Constructor

| Parameter | Type | Required | Default | Purpose |
|-----------|------|----------|---------|---------|
| `content` | `str` | **Yes** | - | Message text |
| `source` | `str` | **Yes** | - | Agent name who sent message |
| `models_usage` | `dict` | No | None | Token usage metadata |

### Example

```python
message = TextMessage(
    content="Here's my analysis...",
    source="analyst"
)
```

---

## HandoffMessage

**Import**:
```python
from autogen_agentchat.messages import HandoffMessage
```

### Constructor

| Parameter | Type | Required | Default | Purpose |
|-----------|------|----------|---------|---------|
| `to` | `str` | **Yes** | - | Recipient agent name |
| `from_agent` | `str` | **Yes** | - | Source agent name |
| `content` | `str` | **Yes** | - | Transfer context |

---

## TerminationCondition

**Import**:
```python
from autogen_agentchat.termination import (
    TerminationCondition,
    MaxMessageTermination,
    StopMessageTermination
)
```

### MaxMessageTermination

| Parameter | Type | Purpose |
|-----------|------|---------|
| `max_messages` | `int` | Stop after this many messages |

**Usage**:
```python
termination = MaxMessageTermination(max_messages=50)
team = RoundRobinGroupChat(
    participants=[...],
    termination_condition=termination
)
```

### StopMessageTermination

| Parameter | Type | Purpose |
|-----------|------|---------|
| `stop_words` | `list[str]` | Stop if message contains these words |

---

## CancellationToken

**Import**:
```python
from autogen_core import CancellationToken
```

### Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `cancel()` | `def cancel()` | Cancel the operation |
| `is_cancelled()` | `def is_cancelled() -> bool` | Check if cancelled |
| `add_callback()` | `def add_callback(callback)` | Register callback |

---

## run() vs run_stream()

| Aspect | run() | run_stream() |
|--------|-------|-------------|
| **Returns** | TaskResult (complete) | AsyncIterator (events) |
| **Blocking** | Waits for completion | Returns immediately |
| **Events** | Not visible | All visible |
| **Use case** | Simple tasks | Observability/debugging |
| **Performance** | Efficient | Slightly overhead |

### Usage Comparison

```python
# run() - Simple
result = await agent.run(task="Research Python")
print(result.messages[-1].content)

# run_stream() - With observability
async for event in agent.run_stream(task="Research Python"):
    print(f"Event: {event}")
```

---

## TaskResult

| Attribute | Type | Purpose |
|-----------|------|---------|
| `messages` | `list[Message]` | All messages in conversation |
| `stop_reason` | `str` | Why execution stopped |
| `metadata` | `dict` | Execution metadata |

### Stop Reasons

- `"max_turns"` - Reached max conversation turns
- `"response"` - Agent responded (not tool call)
- `"max_tool_iterations"` - Exceeded tool call limit
- `"stop_condition"` - Termination condition met
- `"cancelled"` - Execution cancelled

---

## Tool Definition Template

```python
from typing import Annotated

def my_tool(
    param1: Annotated[str, "Description of param1"],
    param2: Annotated[int, "Description of param2"] = 5
) -> str:
    """Tool description.
    
    Longer explanation of what this tool does,
    when to use it, and what it returns.
    """
    # Implementation
    return "result string"
```

### Requirements

- Must return `str`
- All parameters must have type hints
- All parameters should have Annotated descriptions
- Must have docstring

---

## Model Configuration Examples

### OpenAI - GPT-4

```python
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY"),
    temperature=0.7,
    max_tokens=2048
)
```

### OpenAI - GPT-3.5-Turbo (Budget)

```python
model_client = OpenAIChatCompletionClient(
    model="gpt-3.5-turbo",
    api_key=os.getenv("OPENAI_API_KEY"),
    temperature=0.5,
    max_tokens=1024
)
```

### Azure OpenAI

```python
model_client = AzureOpenAIChatCompletionClient(
    model="gpt-4-deployment",
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    api_version="2024-02-15-preview",
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    temperature=0.7
)
```

### For Reproducibility

```python
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY"),
    temperature=0,  # Deterministic
    seed=42  # Reproducible
)
```

---

## AssistantAgent Configuration Examples

### Research Agent

```python
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    system_message="""You are an expert researcher.
- Search for credible sources
- Synthesize findings
- Always cite sources""",
    tools=[web_search],
    max_tool_iterations=5,
    reflect_on_tool_use=True  # Verify findings
)
```

### Code Reviewer

```python
reviewer = AssistantAgent(
    name="code_reviewer",
    model_client=model_client,
    system_message="""You are a senior code reviewer.
- Find bugs and vulnerabilities
- Suggest improvements
- Check performance""",
    tools=[run_tests, analyze_code],
    max_tool_iterations=3
)
```

### Data Analyst

```python
analyst = AssistantAgent(
    name="analyst",
    model_client=model_client,
    system_message="Analyze data and provide insights",
    tools=[process_data, visualize, compute_stats],
    handoffs=["writer"],  # Can hand off to writer
    max_tool_iterations=5
)
```

---

## Team Configuration Examples

### RoundRobin - Fixed Order

```python
team = RoundRobinGroupChat(
    participants=[researcher, analyzer, writer],
    max_turns=10
)
```

### With Termination

```python
from autogen_agentchat.termination import MaxMessageTermination

team = RoundRobinGroupChat(
    participants=[researcher, analyzer, writer],
    termination_condition=MaxMessageTermination(max_messages=50)
)
```

---

## Common Configuration Patterns

### Pattern: Development vs Production

```python
def get_model_client(env: str):
    if env == "production":
        model = "gpt-4"
        temperature = 0.3
    else:
        model = "gpt-3.5-turbo"
        temperature = 0.7
    
    return OpenAIChatCompletionClient(
        model=model,
        api_key=os.getenv("OPENAI_API_KEY"),
        temperature=temperature
    )

model_client = get_model_client(os.getenv("ENV", "development"))
```

### Pattern: Multi-Agent with Different Models

```python
# Expensive model for complex reasoning
complex_model = OpenAIChatCompletionClient(
    model="gpt-4",
    temperature=0.5
)

# Cheaper model for simple tasks
simple_model = OpenAIChatCompletionClient(
    model="gpt-3.5-turbo",
    temperature=0.5
)

researcher = AssistantAgent(
    name="researcher",
    model_client=complex_model  # More capable
)

classifier = AssistantAgent(
    name="classifier",
    model_client=simple_model  # Sufficient for classification
)
```

---

## Error Messages & Meanings

| Error | Cause | Fix |
|-------|-------|-----|
| `Missing API key` | api_key not provided | `api_key=os.getenv("OPENAI_API_KEY")` |
| `Invalid API key` | Wrong or expired key | Check your key in dashboard |
| `Model not found` | Model name incorrect | Use valid model: `gpt-4`, `gpt-3.5-turbo` |
| `Rate limit exceeded` | Too many requests | Reduce concurrent requests |
| `Context length exceeded` | Too many tokens | Reduce max_tokens or limit history |
| `Tool not callable` | Tool not a function | Define as `def my_tool(...):` |
| `Missing type hints` | Tool parameters untyped | Add type hints: `param: str` |
| `Handoff failed` | Target agent not in team | Add target to participants |

---

## Version Notes

- **Current Version**: AutoGen 0.2.x
- **API Stability**: Stable for most parameters
- **Changes from 0.1**: Major architecture overhaul
- **Deprecated**: Legacy APIs from AutoGen < 0.2

---

## Next Steps

- Use this as a reference while reading detailed documentation
- Bookmark for quick parameter lookup
- Check official docs for version-specific details

---

## References

- [Official AutoGen Docs](https://microsoft.github.io/autogen/)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [Azure OpenAI Docs](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
