# Part 24: Components & Configuration

## What It Is

**Components** are reusable, configurable parts. **ComponentConfig** lets you define agents/teams in config files instead of code.

```
Hardcoded:
  agent = AssistantAgent("name", model_client=client, tools=[t1, t2])

Config-based:
  # config.json
  {
    "type": "AssistantAgent",
    "name": "agent",
    "model_client": "...",
    "tools": ["t1", "t2"]
  }
```

---

## Component Model Overview

### What's a Component?

Anything that can be:
- ✅ Serialized (saved to config)
- ✅ Deserialized (loaded from config)
- ✅ Registered (in a registry)

Examples: Agents, Teams, Tools, Model Clients

### Benefits

✅ Separate config from code  
✅ Dynamic construction  
✅ Hot reload configuration  
✅ Large systems management  

---

## ComponentConfig Pattern

### Basic Config

```json
{
  "components": {
    "model_client": {
      "type": "OpenAIClient",
      "model": "gpt-4-turbo",
      "api_key": "${OPENAI_API_KEY}"
    },
    
    "researcher": {
      "type": "AssistantAgent",
      "name": "researcher",
      "model_client": "@model_client",
      "tools": ["@web_search_tool"],
      "system_message": "You are a researcher..."
    },
    
    "analyzer": {
      "type": "AssistantAgent",
      "name": "analyzer",
      "model_client": "@model_client",
      "system_message": "You are an analyst..."
    },
    
    "team": {
      "type": "RoundRobinGroupChat",
      "participants": ["@researcher", "@analyzer"],
      "max_turns": 20
    }
  }
}
```

---

## Configuration-Based Architecture

### Example: Load from Config

```python
import json
from autogen_agentchat import load_config

# Load
with open("config.json", "r") as f:
    config = json.load(f)

# Build components
registry = {}

# 1. Build model client
model_client = OpenAIClient(
    model=config["model_client"]["model"],
    api_key=os.getenv(config["model_client"]["api_key"].strip("${}"))
)
registry["model_client"] = model_client

# 2. Build agents
for name, agent_config in config["agents"].items():
    agent = AssistantAgent(
        name=agent_config["name"],
        model_client=registry[agent_config["model_client"]],
        system_message=agent_config.get("system_message", "")
    )
    registry[name] = agent

# 3. Build team
team = RoundRobinGroupChat(
    participants=[registry[name] for name in config["team"]["participants"]],
    max_turns=config["team"]["max_turns"]
)

# Run
result = await team.run(task="...")
```

---

## Composition Patterns

### Pattern 1: Layered Composition

```json
{
  "components": {
    "layer1": {
      "type": "SimpleAgent"
    },
    "layer2": {
      "type": "TeamOfAgents",
      "participants": ["@layer1"]
    },
    "layer3": {
      "type": "SelectorGroupChat",
      "participants": ["@layer2"]
    }
  }
}
```

### Pattern 2: Multi-Team Architecture

```json
{
  "components": {
    "research_team": {
      "type": "RoundRobinGroupChat",
      "participants": ["@researcher", "@analyzer"]
    },
    
    "review_team": {
      "type": "RoundRobinGroupChat",
      "participants": ["@reviewer", "@approver"]
    },
    
    "orchestrator": {
      "type": "Swarm",
      "participants": ["@research_team", "@review_team"]
    }
  }
}
```

---

## Serialization

### Save Configuration

```python
def save_config(agent, filepath):
    """Save agent config"""
    
    config = {
        "type": "AssistantAgent",
        "name": agent.name,
        "model_client": {
            "type": "OpenAIClient",
            "model": agent.model_client.model
        },
        "tools": [tool.name for tool in agent.tools],
        "system_message": agent.system_message
    }
    
    with open(filepath, "w") as f:
        json.dump(config, f, indent=2)

# Usage
save_config(agent, "agent_config.json")
```

### Load Configuration

```python
def load_config(filepath):
    """Load agent config"""
    
    with open(filepath, "r") as f:
        config = json.load(f)
    
    # Reconstruct
    agent = AssistantAgent(
        name=config["name"],
        model_client=OpenAIClient(model=config["model_client"]["model"]),
        system_message=config.get("system_message", "")
    )
    
    return agent

# Usage
agent = load_config("agent_config.json")
```

---

## Dynamic Construction

### Factory Pattern

```python
COMPONENT_REGISTRY = {
    "AssistantAgent": AssistantAgent,
    "UserProxyAgent": UserProxyAgent,
    "CustomAgent": CustomAgent,
    "RoundRobinGroupChat": RoundRobinGroupChat,
    "SelectorGroupChat": SelectorGroupChat,
    "Swarm": Swarm,
}

def create_component(config):
    """Create component from config"""
    
    component_type = config["type"]
    
    if component_type not in COMPONENT_REGISTRY:
        raise ValueError(f"Unknown component: {component_type}")
    
    ComponentClass = COMPONENT_REGISTRY[component_type]
    
    # Build with config params
    return ComponentClass(**config.get("params", {}))

# Usage
config = {
    "type": "AssistantAgent",
    "params": {
        "name": "analyzer",
        "model_client": model_client
    }
}
agent = create_component(config)
```

---

## Registry Patterns

### Simple Registry

```python
class ComponentRegistry:
    """Manage components"""
    
    def __init__(self):
        self.components = {}
    
    def register(self, name: str, component):
        """Register component"""
        self.components[name] = component
    
    def get(self, name: str):
        """Retrieve component"""
        return self.components.get(name)
    
    def list_all(self):
        """List all components"""
        return list(self.components.keys())

# Usage
registry = ComponentRegistry()
registry.register("agent_a", AssistantAgent("a", ...))
registry.register("agent_b", AssistantAgent("b", ...))

agent = registry.get("agent_a")
```

---

## Benefits for Large Systems

### Example: 100-Agent System

Without config:
```python
# Thousands of lines of hardcoded agent creation
agent1 = AssistantAgent("1", ...)
agent2 = AssistantAgent("2", ...)
# ... x100
```

With config:
```json
{
  "agents": [
    {"name": "1", "type": "AssistantAgent"},
    {"name": "2", "type": "AssistantAgent"},
    // ... x100 (much cleaner)
  ]
}
```

Then load:
```python
agents = [create_component(cfg) for cfg in config["agents"]]
```

---

## Common Patterns

### Pattern: Environment-Specific Config

```json
{
  "development": {
    "model": "gpt-3.5-turbo",
    "max_turns": 5,
    "debug": true
  },
  "production": {
    "model": "gpt-4",
    "max_turns": 20,
    "debug": false
  }
}
```

Usage:
```python
env = os.getenv("ENV", "development")
config = load_config(f"config_{env}.json")
```

### Pattern: Template Config

```python
def create_standard_team(config):
    """Create team from minimal config"""
    
    config_template = {
        "team_type": "RoundRobin",
        "agents": 3,
        "model": "gpt-4"
    }
    config_template.update(config)
    
    # Build based on template
    agents = [
        AssistantAgent(f"agent_{i}", ...)
        for i in range(config_template["agents"])
    ]
    
    return RoundRobinGroupChat(participants=agents)
```

---

## Common Mistakes

### Mistake 1: Circular References

```json
{
  "agent_a": {
    "dependency": "@agent_b"
  },
  "agent_b": {
    "dependency": "@agent_a"  // Circular!
  }
}
```

**Fix**: Ensure acyclic dependency graph

### Mistake 2: Missing References

```json
{
  "agent": {
    "model_client": "@missing_client"  // Doesn't exist!
  }
}
```

**Fix**: Validate all references exist

### Mistake 3: Type Mismatch

```json
{
  "team": {
    "type": "RoundRobinGroupChat",
    "max_turns": "20"  // String instead of int!
  }
}
```

**Fix**: Validate types during loading

---

## When to Use Components

✅ **Use config**:
- Multiple environments
- Large systems (50+ agents)
- Frequent reconfigurations
- Non-technical users need to adjust

❌ **Use code**:
- Simple systems
- One-off scripts
- Rapid prototyping
- Complex logic

---

## Next Steps

- [23-autogen-core.md](23-autogen-core.md) - Low-level runtime
- [25-complete-project.md](25-complete-project.md) - Full example
- Official AutoGen: https://microsoft.github.io/autogen/

---

## References

- Official AutoGen: https://microsoft.github.io/autogen/
