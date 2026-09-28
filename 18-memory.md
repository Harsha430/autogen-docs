# Part 18: Memory Systems

## What It Is

**Memory** lets agents remember facts across different conversations. Unlike context (temporary), memory persists.

```
Conversation 1: Learn "Alice is CEO"
  ↓ (Save to memory)
Conversation 2: Remember "Alice is CEO"
  ↓ (Retrieved from memory)
Conversation 3: Use remembered fact
```

---

## Memory Types

### Episodic Memory
**What**: Specific events/conversations

```
"On Jan 1, user asked about Q4 budget"
"On Jan 2, Alice approved the plan"
```

**When useful**: Timeline of decisions

### Semantic Memory
**What**: General facts/knowledge

```
"Alice is CEO"
"Q4 budget is $500K"
"Project deadline: March 31"
```

**When useful**: Reference information

### Procedural Memory
**What**: How to do things

```
"To approve budget: 1) Request submitted, 2) Alice reviews, 3) Approved"
```

**When useful**: Workflow steps

---

## Adding Memories During Execution

```python
from autogen_agentchat.agents import AssistantAgent

class MemoryAgent(BaseChatAgent):
    """Agent that learns and recalls facts"""
    
    def __init__(self, name, model_client=None):
        self._name = name
        self.model_client = model_client
        self.memory = {}  # Semantic memory store
    
    @property
    def name(self):
        return self._name
    
    async def on_messages(self, messages, sender):
        """Process and update memory"""
        
        latest = messages[-1].content
        
        # Learn from messages
        if "is" in latest and "=" not in latest:
            # Simple pattern: "X is Y"
            parts = latest.split(" is ")
            if len(parts) == 2:
                key, value = parts
                self.memory[key.strip()] = value.strip()
                print(f"Remembered: {key.strip()} = {value.strip()}")
        
        # Use memory
        response_text = "Current facts: " + str(self.memory)
        
        response = TextMessage(
            content=response_text,
            source=self.name
        )
        return [response], "processed"
```

---

## Retrieving Relevant Memories

### Simple Lookup

```python
def recall_memory(agent, query: str):
    """Retrieve fact from memory"""
    for key, value in agent.memory.items():
        if query.lower() in key.lower():
            return value
    return None

# Usage
value = recall_memory(agent, "CEO")  # Returns "Alice"
```

### Semantic Search

```python
from difflib import SequenceMatcher

def find_similar_memory(agent, query: str):
    """Find closest matching memory"""
    
    best_match = None
    best_score = 0.5
    
    for key, value in agent.memory.items():
        score = SequenceMatcher(None, query.lower(), key.lower()).ratio()
        if score > best_score:
            best_match = (key, value)
            best_score = score
    
    return best_match

# Usage
match = find_similar_memory(agent, "Who's the CEO?")  # Finds "Alice is CEO"
```

---

## Memory vs Context: Key Differences

| Aspect | Memory | Context |
|--------|--------|---------|
| **Scope** | Across conversations | Current conversation |
| **Lifespan** | Permanent (until deleted) | Temporary (one run) |
| **Storage** | Persistent (file, DB) | Volatile (RAM) |
| **Size** | Unlimited (good indexing) | Limited (token window) |
| **Purpose** | Facts to reuse | Messages for LLM |

```python
# CONTEXT - Just for current conversation
context = result.messages  # These are temporary

# MEMORY - Persisted facts
memory = {
    "user_name": "Alice",
    "last_budget": "$500K"
}
# This persists to next conversation
```

---

## Practical Memory Implementations

### File-Based Memory

```python
import json
import os

class FileMemory:
    def __init__(self, filepath="memory.json"):
        self.filepath = filepath
        self.memory = self._load()
    
    def _load(self):
        if os.path.exists(self.filepath):
            with open(self.filepath, "r") as f:
                return json.load(f)
        return {}
    
    def _save(self):
        with open(self.filepath, "w") as f:
            json.dump(self.memory, f, indent=2)
    
    def remember(self, key: str, value: str):
        """Store fact"""
        self.memory[key] = value
        self._save()
    
    def recall(self, key: str):
        """Retrieve fact"""
        return self.memory.get(key)

# Usage
memory = FileMemory()
memory.remember("project_deadline", "March 31")
print(memory.recall("project_deadline"))  # "March 31"
```

### Database Memory

```python
# Pseudo-code for database memory
class DatabaseMemory:
    def __init__(self, connection):
        self.db = connection
    
    def remember(self, key: str, value: str, source: str = ""):
        """Store in database"""
        self.db.execute(
            "INSERT INTO memories (key, value, source) VALUES (?, ?, ?)",
            (key, value, source)
        )
        self.db.commit()
    
    def recall(self, key: str):
        """Retrieve from database"""
        result = self.db.execute(
            "SELECT value FROM memories WHERE key = ?",
            (key,)
        ).fetchone()
        return result[0] if result else None
    
    def search(self, query: str):
        """Search memories"""
        results = self.db.execute(
            "SELECT key, value FROM memories WHERE key LIKE ?",
            (f"%{query}%",)
        ).fetchall()
        return results
```

---

## Workflow: Learn and Recall

```python
class LearningAgent(BaseChatAgent):
    """Agent that learns from conversation"""
    
    def __init__(self, name, model_client, memory_file="agent_memory.json"):
        self._name = name
        self.model_client = model_client
        self.memory = FileMemory(memory_file)
    
    @property
    def name(self):
        return self._name
    
    async def on_messages(self, messages, sender):
        """Learn facts, use memory"""
        
        latest = messages[-1].content
        
        # Phase 1: Extract facts to remember
        # (Simple: look for "remember: X"pattern)
        if "remember:" in latest.lower():
            fact = latest.split("remember:")[-1].strip()
            self.memory.remember(f"fact_{len(messages)}", fact)
        
        # Phase 2: Recall relevant facts
        recalled = []
        for key, value in self.memory.memory.items():
            if any(word in latest.lower() for word in value.lower().split()):
                recalled.append(f"- {value}")
        
        # Phase 3: Respond with memory
        memory_context = "\n".join(recalled) if recalled else "No relevant memories"
        
        response = TextMessage(
            content=f"Based on memory:\n{memory_context}\n\nProcessing new request...",
            source=self.name
        )
        return [response], "processed"
```

---

## When Memory Is Worth It

**Worth implementing**:
- Multi-day conversations with same user
- Production workflows needing context
- Learning-focused systems
- Long-running agents

**Not worth it**:
- One-off tasks
- No multi-turn value
- Short conversations
- Research/testing only

---

## Common Mistakes

### Mistake 1: Confusing Memory and Context
```python
# WRONG - Treating context as memory
result = await agent.run(task="A")
messages_a = result.messages

result = await agent.run(task="B")
messages_b = result.messages

# messages_a lost! Different run = different context

# RIGHT - Save important facts to memory
result = await agent.run(task="A")
# Extract fact
if "budget" in str(result.messages):
    memory.remember("budget", "$500K")

result = await agent.run(task="B")
# Recall fact
budget = memory.recall("budget")
```

### Mistake 2: Memory Explosion
```python
# WRONG - Store everything
for msg in result.messages:
    memory.remember(f"msg_{i}", msg.content)  # Stores 1000s of messages!

# RIGHT - Store only key facts
if "decision:" in msg.content:
    memory.remember("decision", msg.content)
```

### Mistake 3: Memory Never Updated
```python
# WRONG - Store once, never refresh
memory.remember("price", "$100")
# Later, price changes to $150
# But memory still says $100!

# RIGHT - Update memory
if new_price := extract_price(latest_msg):
    memory.remember("price", new_price)  # Overwrite
```

---

## Memory Decay Pattern

```python
import time
from datetime import datetime, timedelta

class DecayingMemory:
    """Memory that expires"""
    
    def __init__(self):
        self.memory = {}  # {key: (value, timestamp)}
    
    def remember(self, key: str, value: str, ttl_days: int = 30):
        """Store with expiration"""
        expiry = datetime.now() + timedelta(days=ttl_days)
        self.memory[key] = (value, expiry)
    
    def recall(self, key: str):
        """Retrieve only if not expired"""
        if key not in self.memory:
            return None
        
        value, expiry = self.memory[key]
        if datetime.now() > expiry:
            del self.memory[key]  # Expired, remove
            return None
        
        return value
```

---

## Next Steps

- [17-model-context.md](17-model-context.md) - Context management
- [19-state-persistence.md](19-state-persistence.md) - Saving state
- [21-observability.md](21-observability.md) - Monitoring memory usage

---

## References

- Official AutoGen: https://microsoft.github.io/autogen/
