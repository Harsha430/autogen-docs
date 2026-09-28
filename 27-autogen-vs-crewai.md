# Part 27: AutoGen vs CrewAI

## At a Glance

| Aspect | AutoGen | CrewAI |
|--------|---------|--------|
| **Origin** | Microsoft Research | Community-driven |
| **Design** | Flexible agents + teams | Structured crew of roles |
| **API Simplicity** | Medium | Simpler |
| **Customization** | High | Medium |
| **Use case** | Research/complex systems | Business workflows |
| **Community** | Smaller, research-focused | Growing, business-friendly |

---

## Conceptual Mapping

```
AutoGen                 CrewAI
├── Agent          ←→  ├── Agent (with Role)
├── AssistantAgent ←→  ├── Crew
├── Team           ←→  ├── Task
├── Tools          ←→  ├── Tools
└── Messages       ←→  └── Tool Results
```

---

## Agent Definition

### AutoGen: Flexible

```python
from autogen_agentchat.agents import AssistantAgent

agent = AssistantAgent(
    name="researcher",
    model_client=OpenAIClient(model="gpt-4"),
    system_message="You are a researcher...",
    tools=[search_tool, verify_tool]
)
```

**Pros**: Maximum flexibility  
**Cons**: More code needed

### CrewAI: Role-Based

```python
from crewai import Agent

researcher = Agent(
    role="Researcher",
    goal="Find accurate information",
    backstory="Expert researcher with 20 years experience",
    tools=[search_tool],
    llm=llm
)
```

**Pros**: Simpler API, clear roles  
**Cons**: Less customization

---

## Team/Crew Definition

### AutoGen: Explicit Orchestration

```python
from autogen_agentchat.teams import RoundRobinGroupChat

team = RoundRobinGroupChat(
    participants=[researcher, analyzer, writer],
    max_turns=10
)

result = await team.run(task="Write article")
```

**Define**:
- Which agents participate
- In what order
- How long to run

### CrewAI: Task-Based

```python
from crewai import Crew, Task

research_task = Task(
    description="Research the topic",
    agent=researcher
)

analysis_task = Task(
    description="Analyze findings",
    agent=analyzer
)

crew = Crew(
    agents=[researcher, analyzer],
    tasks=[research_task, analysis_task]
)

result = crew.kickoff()
```

**Define**:
- Tasks (goals)
- Which agent does each
- Dependencies implicit

---

## Tool Usage

### AutoGen: Agents Call Tools

```python
# Tools defined separately
def search(query: str) -> str:
    """Search"""
    return "..."

# Agent uses tools
agent = AssistantAgent(
    name="researcher",
    tools=[Tool(name="search", func=search)],
    model_client=client
)

# Agent decides when to call
result = await agent.run(task="Search for X")
# Agent: "I'll search for that"
# [Calls search tool]
```

### CrewAI: Structured Tool Calling

```python
from crewai_tools import tool

@tool
def search(query: str) -> str:
    """Search the web"""
    return "..."

agent = Agent(
    role="Researcher",
    tools=[search]
)

# CrewAI manages tool calling
crew = Crew(agents=[agent], tasks=[task])
```

---

## Message Flow

### AutoGen: Explicit Messages

```python
# Every message visible
result = await team.run(task="...")

for msg in result.messages:
    print(f"{msg.source}: {msg.content}")

# Output:
# researcher: I found 3 sources...
# analyzer: These sources suggest...
# writer: The article is ready
```

### CrewAI: Implicit Flow

```python
# See task results
crew = Crew(agents=[...], tasks=[research_task, analysis_task])
result = crew.kickoff()

# Output:
# Task: Research the topic
# Result: Found 3 sources...
# 
# Task: Analyze findings
# Result: Analysis complete
```

---

## Routing & Control

### AutoGen: Flexible Routing

```python
# Option 1: Fixed order
team = RoundRobinGroupChat(participants=[A, B, C])

# Option 2: Selector chooses
team = SelectorGroupChat(participants=[A, B, C])

# Option 3: Agents handoff
# Swarm(participants=[A, B, C])
```

### CrewAI: Task-Based Routing

```python
# Tasks run in sequence by default
task1 = Task(description="...", agent=researcher)
task2 = Task(description="...", agent=analyzer)

# Each task's output feeds to next
# Deterministic flow
```

---

## Customization

### AutoGen: Deep Customization

```python
class MyCustomAgent(BaseChatAgent):
    """Create completely custom agent"""
    
    async def on_messages(self, messages, sender):
        # Your logic here
        return [response], "ok"

# Use with any team
team.participants = [MyCustomAgent(...), agent_b]
```

**Flexibility**: Can do anything  
**Complexity**: Lots of code

### CrewAI: Limited Customization

```python
# Extend Agent class
class CustomAgent(Agent):
    def think(self):
        # Override thinking process
        pass

# More limited, but simpler
```

---

## Memory & Context

### AutoGen: Message-Based

```python
# All messages are context
result = await team.run(...)

# Full history available
for msg in result.messages:
    if "important" in msg.content:
        print(msg)
```

### CrewAI: Task-Based Memory

```python
# Memory per task
# Crew maintains working memory
# Less explicit, more implicit
```

---

## When to Use Each

### Use AutoGen If:

✅ Need maximum flexibility  
✅ Complex agent interactions  
✅ Research/experimentation  
✅ Multi-team systems  
✅ Custom routing logic  

**Examples**:
- Multi-agent research system
- Autonomous agent networks
- Complex orchestration

### Use CrewAI If:

✅ Want simpler API  
✅ Business workflows  
✅ Clear sequential tasks  
✅ Role-based agents  
✅ Learning AutoGen too complex  

**Examples**:
- Data analysis workflow
- Report generation
- Business process automation

---

## Example: Same Task

### AutoGen

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat

team = RoundRobinGroupChat(
    participants=[
        AssistantAgent("researcher", client),
        AssistantAgent("analyzer", client),
        AssistantAgent("writer", client)
    ],
    max_turns=12
)

result = await team.run(task="Write analysis")
```

**Lines**: ~12  
**Flexibility**: High  
**Clarity**: Medium

### CrewAI

```python
from crewai import Agent, Crew, Task

researcher = Agent(role="Researcher", tools=[search])
analyzer = Agent(role="Analyzer")
writer = Agent(role="Writer")

tasks = [
    Task(description="Research", agent=researcher),
    Task(description="Analyze", agent=analyzer),
    Task(description="Write", agent=writer)
]

crew = Crew(agents=[researcher, analyzer, writer], tasks=tasks)
result = crew.kickoff()
```

**Lines**: ~20  
**Flexibility**: Medium  
**Clarity**: High

---

## Key Differences Summary

| Feature | AutoGen | CrewAI |
|---------|---------|--------|
| **Learning curve** | Medium | Low |
| **Flexibility** | High | Medium |
| **API surface** | Large | Small |
| **Message visibility** | High | Low |
| **State management** | Explicit | Implicit |
| **Customization** | Deep | Limited |
| **Documentation** | Research-heavy | Business-focused |
| **Community size** | Smaller | Growing |
| **Production use** | Proven | Emerging |

---

## Common Migration Issues

### Issue 1: Implicit vs Explicit

```python
# CrewAI (implicit)
result = crew.kickoff()  # Magic inside

# AutoGen (explicit)
result = await team.run(task=...)  # See everything
```

### Issue 2: Message Access

```python
# AutoGen: Full messages
for msg in result.messages:
    print(msg.content)

# CrewAI: Task results only
# Limited direct message access
```

### Issue 3: Tool Control

```python
# AutoGen: Agent decides when to call tools
# Full agentic loop

# CrewAI: Structured tool calling
# Less autonomy for agent
```

---

## Hybrid Approaches

### Use Both Together

```python
# CrewAI for sequential workflow
# AutoGen for agent interactions

# Step 1: CrewAI tasks
crew_result = crew.kickoff()

# Step 2: AutoGen team for complex negotiation
team_result = await team.run(task=crew_result)
```

---

## Choosing: Decision Tree

```
Does your workflow have clear sequential steps?
  ├─ YES → CrewAI (simpler)
  └─ NO → AutoGen (more flexible)

Do you need maximum customization?
  ├─ YES → AutoGen
  └─ NO → CrewAI

Is your team business-focused?
  ├─ YES → CrewAI (simpler API)
  └─ NO → AutoGen (research-friendly)

New to multi-agent systems?
  ├─ YES → CrewAI (gentler learning curve)
  └─ NO → AutoGen (more powerful)
```

---

## Pros & Cons Comparison

### AutoGen

**Pros**:
- Maximum flexibility
- Full control over agents and routing
- Better for research
- Explicit message history

**Cons**:
- More verbose
- Steeper learning curve
- More code required
- Research documentation

### CrewAI

**Pros**:
- Simpler API
- Business-friendly
- Easier to learn
- Clear role definitions

**Cons**:
- Less flexibility
- Less customization
- Smaller community
- Emerging platform (less mature)

---

## Next Steps

- Choose based on your use case
- Start with simpler framework if learning
- Move to more powerful if needed
- Consider hybrid approach

---

## References

- AutoGen: https://microsoft.github.io/autogen/
- CrewAI: https://docs.crewai.com/
