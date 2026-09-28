# Part 3: Model Clients - LLM Provider Abstraction

## Why Model Clients Exist

### Problem Without Model Client

```python
# Agent tightly coupled to OpenAI
class ResearchAgent:
    def __init__(self):
        self.openai_client = OpenAI(api_key="...")
        self.model = "gpt-4"
        self.temperature = 0.7
    
    async def research(self, topic):
        response = await self.openai_client.chat.completions.create(
            model=self.model,
            messages=[...],
            temperature=self.temperature,
            tools=[...],
            # OpenAI-specific parameters scattered everywhere
        )
        # OpenAI-specific response parsing
        return response.choices[0].message.content

# Problems:
# - Switching to Azure requires rewriting agent
# - Switching to Claude requires rewriting agent
# - Agent logic mixed with provider logic
# - Difficult to test with mock LLM
```

### Solution With Model Client

```python
# Agent uses abstraction
class ResearchAgent:
    def __init__(self, model_client):
        self.model_client = model_client  # Generic interface
    
    async def research(self, topic):
        response = await self.model_client.create(
            messages=[...],
            tools=[...],
        )
        # Generic response handling
        return response.choices[0].message.content

# Usage:
# With OpenAI
model_client = OpenAIChatCompletionClient(model="gpt-4")
agent = ResearchAgent(model_client)

# With Azure (same agent code)
model_client = AzureOpenAIChatCompletionClient(model="gpt-4")
agent = ResearchAgent(model_client)

# With mock for testing
model_client = MockChatClient()
agent = ResearchAgent(model_client)
```

## Model Client Architecture

```
┌─────────────────────────────────┐
│  Agents (AssistantAgent, etc)   │
│                                 │
│ Do NOT know about:              │
│ - LLM provider details          │
│ - API specifics                 │
│ - Authentication methods        │
│                                 │
│ Only know:                      │
│ - Call model_client.create()    │
│ - Receive generic response      │
└────────────────┬────────────────┘
                 │
┌────────────────▼────────────────┐
│  Model Client Interface         │
│                                 │
│  async def create(              │
│    messages,                    │
│    tools=None,                  │
│    ...                          │
│  )                              │
│                                 │
│  Generic interface that all     │
│  implementations follow         │
└────────────────┬────────────────┘
                 │
    ┌────────────┴────────────────┐
    │                             │
┌───▼──────────────────┐ ┌───────▼──────────────┐
│ OpenAIChatCompletion │ │ AzureOpenAIChat      │
│ Client               │ │ CompletionClient     │
│                      │ │                      │
│ Implements:          │ │ Implements:          │
│ - create()           │ │ - create()           │
│ - Calls openai.com   │ │ - Calls Azure API    │
│ - Handles response   │ │ - Handles response   │
└──────────────────────┘ └──────────────────────┘
```

---

## Current Model Client Options

### 1. OpenAIChatCompletionClient

The most common model client for OpenAI's API.

#### Import
```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
```

#### Constructor
```python
model_client = OpenAIChatCompletionClient(
    model="gpt-4",              # Model name
    api_key=os.getenv("OPENAI_API_KEY"),  # API key
    
    # Optional
    base_url="https://api.openai.com/v1",  # Custom endpoint
    timeout=60,                 # Request timeout in seconds
    temperature=0.7,            # Randomness (0-2)
    max_tokens=4096,            # Max output tokens
    top_p=0.95,                 # Nucleus sampling
    seed=None,                  # For deterministic outputs
    extra_create_args={         # Additional OpenAI params
        "presence_penalty": 0,
        "frequency_penalty": 0,
    }
)
```

#### Key Parameters Explained

| Parameter | Type | Purpose | Default | Example |
|-----------|------|---------|---------|---------|
| `model` | str | Which LLM to use | Required | `"gpt-4"`, `"gpt-4-turbo"` |
| `api_key` | str | OpenAI API key | Required | `os.getenv("OPENAI_API_KEY")` |
| `base_url` | str | API endpoint (for proxies) | OpenAI default | Custom domain |
| `timeout` | int | Request timeout in seconds | 300 | 60, 120 |
| `temperature` | float | Response randomness | 1.0 | 0 (deterministic), 1.5 (creative) |
| `max_tokens` | int | Maximum output tokens | None (unlimited) | 2048 |
| `top_p` | float | Nucleus sampling parameter | 1.0 | 0.9 |
| `seed` | int or None | For reproducible outputs | None | 42 |
| `extra_create_args` | dict | Additional OpenAI parameters | {} | `{"presence_penalty": 0.5}` |

#### Usage Example
```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_agentchat.agents import AssistantAgent

# Setup
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY"),
    temperature=0.7,
    max_tokens=2048
)

# Use with agent
agent = AssistantAgent(
    name="researcher",
    model_client=model_client,
    system_message="You are a research expert."
)

# Agent will automatically use this model client
# for all LLM calls
```

#### Common Mistakes

❌ **Mistake 1: Hardcoding API key**
```python
# WRONG
model_client = OpenAIChatCompletionClient(
    api_key="sk-12345..."  # Exposed in code!
)
```

✅ **Correct**
```python
# RIGHT
model_client = OpenAIChatCompletionClient(
    api_key=os.getenv("OPENAI_API_KEY")
)
```

❌ **Mistake 2: Forgetting API key**
```python
# WRONG - Will fail with auth error
model_client = OpenAIChatCompletionClient(model="gpt-4")
```

✅ **Correct**
```python
# RIGHT
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY")
)
```

---

### 2. AzureOpenAIChatCompletionClient

For deploying with Azure OpenAI Service.

#### Import
```python
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
```

#### Constructor
```python
model_client = AzureOpenAIChatCompletionClient(
    model="gpt-4",              # Deployment name
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    api_version="2024-02-15-preview",  # Azure API version
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    
    # Optional (same as OpenAI client)
    timeout=60,
    temperature=0.7,
    max_tokens=4096,
)
```

#### Key Parameters

| Parameter | Required | Purpose |
|-----------|----------|---------|
| `model` | Yes | Azure deployment name |
| `api_key` | Yes | Azure OpenAI API key |
| `api_version` | Yes | Azure API version |
| `azure_endpoint` | Yes | Azure resource endpoint |

#### Azure Setup Example
```python
import os
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
from autogen_agentchat.agents import AssistantAgent

# Environment variables (set in .env or system)
# AZURE_OPENAI_API_KEY=your-key
# AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
# AZURE_OPENAI_DEPLOYMENT_NAME=your-deployment

model_client = AzureOpenAIChatCompletionClient(
    model=os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME", "gpt-4"),
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    api_version="2024-02-15-preview",
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    temperature=0.7
)

agent = AssistantAgent(
    name="azure_researcher",
    model_client=model_client
)
```

---

## Model Configuration Best Practices

### 1. Parameterize Everything

```python
# WRONG - hardcoded config
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key="sk-...",
    temperature=0.7,
    max_tokens=2048
)

# RIGHT - configuration from environment
MODEL_CONFIG = {
    "model": os.getenv("LLM_MODEL", "gpt-4"),
    "api_key": os.getenv("OPENAI_API_KEY"),
    "temperature": float(os.getenv("LLM_TEMPERATURE", "0.7")),
    "max_tokens": int(os.getenv("LLM_MAX_TOKENS", "2048")),
}

model_client = OpenAIChatCompletionClient(**MODEL_CONFIG)
```

### 2. Create Model Client Once (Reuse)

```python
# WRONG - creates new client each time
async def analyze_data(data):
    model_client = OpenAIChatCompletionClient(model="gpt-4")
    agent = AssistantAgent(model_client=model_client)
    return await agent.run(task=data)

# RIGHT - create once, reuse
model_client = OpenAIChatCompletionClient(model="gpt-4")

async def analyze_data(data):
    agent = AssistantAgent(model_client=model_client)
    return await agent.run(task=data)
```

### 3. Use Type Hints

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient

def create_agent(
    model_client: OpenAIChatCompletionClient,
    name: str
) -> AssistantAgent:
    return AssistantAgent(
        name=name,
        model_client=model_client
    )
```

---

## Model Parameters Deep Dive

### temperature (Randomness)

Controls how creative vs deterministic the model is.

```
0.0   = Deterministic (always same answer)
0.7   = Balanced (default for most tasks)
1.5+  = Very creative (varied outputs)
2.0   = Maximum (may be less coherent)
```

**When to adjust**:
- **Lower (0-0.3)**: Fact-based tasks (research, data extraction)
- **Default (0.7)**: General tasks
- **Higher (1.2-2.0)**: Creative tasks (brainstorming, writing)

```python
# Fact-based research
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    temperature=0.2  # Consistent, factual
)

# Creative writing
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    temperature=1.2  # More variety
)
```

### max_tokens (Output Limit)

Maximum tokens in the response.

```
None = No limit (use model's maximum)
1024 = 1K tokens (~750 words)
2048 = 2K tokens (~1500 words)
4096 = 4K tokens (~3000 words)
```

**Important**: This limits OUTPUT, not context window.

```python
# Short responses
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    max_tokens=256  # ~150 words
)

# Long-form output
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    max_tokens=4096  # ~3000 words
)
```

### seed (Reproducibility)

Makes responses deterministic (same seed = same response).

```python
# Reproducible results
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    seed=42,  # Same seed = consistent output
    temperature=0.0  # Should use with low temperature
)

# Non-reproducible (default)
model_client = OpenAIChatCompletionClient(
    model="gpt-4"
    # seed not set
)
```

**When to use**:
- Testing and debugging
- Deterministic batch processing
- Educational scenarios

---

## Advanced Configuration

### Custom Endpoints (Proxies)

```python
# Using a proxy or custom endpoint
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url="https://your-proxy.example.com/v1"  # Custom endpoint
)
```

### Extra Parameters

Pass additional parameters to the underlying API:

```python
model_client = OpenAIChatCompletionClient(
    model="gpt-4",
    extra_create_args={
        "presence_penalty": 0.5,      # Reduce repetition
        "frequency_penalty": 0.5,     # Reduce topic repetition
        "top_p": 0.9,                 # Nucleus sampling
    }
)
```

---

## Model Selection Guide

### For General Tasks
```
gpt-4           = Most capable (best for complex reasoning)
gpt-4-turbo     = Faster than gpt-4, good for most tasks
gpt-3.5-turbo   = Cheapest, good for simple tasks
```

### For Cost-Sensitive Tasks
```
gpt-3.5-turbo   = Lowest cost (~5x cheaper than gpt-4)
                  Trade-off: Less capable
```

### For Speed
```
gpt-4-turbo     = ~2x faster than gpt-4
                  Trade-off: Slightly less capable
```

### For Quality
```
gpt-4           = Best quality
                  Trade-off: More expensive, slower
```

---

## Common Configuration Patterns

### Pattern 1: Development vs Production

```python
import os

def get_model_client():
    if os.getenv("ENVIRONMENT") == "production":
        return OpenAIChatCompletionClient(
            model="gpt-4",
            temperature=0.3,  # More conservative
            max_tokens=2048,
            timeout=120
        )
    else:  # Development
        return OpenAIChatCompletionClient(
            model="gpt-3.5-turbo",  # Cheaper for dev
            temperature=0.7,
            max_tokens=1024,
            timeout=60
        )

model_client = get_model_client()
```

### Pattern 2: Task-Specific Clients

```python
# Research task - deterministic
research_client = OpenAIChatCompletionClient(
    model="gpt-4",
    temperature=0.2,
    max_tokens=2048
)

# Creative task - varied
creative_client = OpenAIChatCompletionClient(
    model="gpt-4",
    temperature=1.2,
    max_tokens=4096
)

research_agent = AssistantAgent(
    name="researcher",
    model_client=research_client
)

writer_agent = AssistantAgent(
    name="writer",
    model_client=creative_client
)
```

### Pattern 3: Cost Optimization

```python
def get_model_client(complexity: str):
    if complexity == "simple":
        model = "gpt-3.5-turbo"
        temperature = 0.5
    elif complexity == "moderate":
        model = "gpt-4-turbo"
        temperature = 0.7
    else:  # complex
        model = "gpt-4"
        temperature = 0.7
    
    return OpenAIChatCompletionClient(
        model=model,
        temperature=temperature,
        max_tokens=2048
    )

# Use based on task complexity
simple_client = get_model_client("simple")
complex_client = get_model_client("complex")
```

---

## Monitoring Model Client Usage

### Getting Token Usage

```python
from autogen_agentchat.teams import RoundRobinGroupChat

team = RoundRobinGroupChat(
    participants=[agent1, agent2],
    max_turns=5
)

result = await team.run(task="Research Python")

# Check token usage
if result.messages:
    last_message = result.messages[-1]
    # Token usage available in metadata
    print(f"Stop reason: {last_message.source}")
```

### Tracking Across Multiple Calls

```python
total_tokens = 0
total_cost = 0

for task in tasks:
    result = await agent.run(task=task)
    
    # Token tracking depends on model_client implementation
    # Check your model_client's get_usage() method
    usage = model_client.get_usage()  # May not be implemented
    if usage:
        total_tokens += usage.total_tokens
        
        # Calculate cost (example: GPT-4)
        input_cost = usage.prompt_tokens * 0.00003
        output_cost = usage.completion_tokens * 0.00006
        total_cost += input_cost + output_cost

print(f"Total tokens: {total_tokens}")
print(f"Total cost: ${total_cost:.2f}")
```

---

## Error Handling

### Common Model Client Errors

```python
import os
from autogen_ext.models.openai import OpenAIChatCompletionClient

try:
    model_client = OpenAIChatCompletionClient(
        model="gpt-4",
        api_key=os.getenv("OPENAI_API_KEY"),  # May be None
        timeout=5  # Short timeout for demo
    )
except Exception as e:
    print(f"Failed to create model client: {e}")

# Common issues:
# - Missing API key environment variable
# - Invalid API key
# - Network timeout
# - Rate limiting
# - Model not available
```

### Retry Logic

```python
import asyncio
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def create_with_retry(max_retries=3):
    for attempt in range(max_retries):
        try:
            model_client = OpenAIChatCompletionClient(model="gpt-4")
            # Verify by making a small request
            return model_client
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Attempt {attempt + 1} failed, retrying in {wait_time}s")
                await asyncio.sleep(wait_time)
            else:
                raise

model_client = await create_with_retry()
```

---

## Next Steps

Now that you understand model clients, move to:
- **[04-assistant-agent.md](04-assistant-agent.md)** to learn how agents use model clients
- Or **[09-tools.md](09-tools.md)** to understand what agents call with model client responses

---

## References

- [OpenAI Chat Completion API](https://platform.openai.com/docs/guides/gpt)
- [Azure OpenAI Service](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [AutoGen Model Client Documentation](https://microsoft.github.io/autogen/docs/topics/models/)

---

## Version Notes

- **Current Version**: AutoGen 0.2.x
- **OpenAI API**: Latest (2024-x-x preview versions supported)
- **Azure API**: Supports recent API versions
- **Custom Clients**: Can be implemented by extending base client interface
