# Part 25: Complete Project - End-to-End Example

## Project: Multi-Stage Research System

This example shows a complete workflow: research → analysis → writing → review → approval → persistence.

---

## Stage 1: Setup & Basic Agent

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.clients import OpenAIClient
import os

# Initialize model client
model_client = OpenAIClient(
    model="gpt-4-turbo",
    api_key=os.getenv("OPENAI_API_KEY")
)

# Create basic researcher agent
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    system_message="""
    You are a research expert. Your task:
    1. Search for reliable information
    2. Verify sources
    3. Summarize findings
    Keep responses concise and factual.
    """
)

# Test
result = await researcher.run(task="What are the latest AI trends?")
print(result.messages[-1].content)
```

---

## Stage 2: Add Tools

```python
from autogen_agentchat.tools import Tool
from autogen_agentchat.agents import AssistantAgent

# Define tools
def web_search(query: str) -> str:
    """Search the web for information"""
    # Simulated search
    return f"Found 100 results for: {query}"

def verify_source(url: str) -> bool:
    """Verify if source is credible"""
    return "https://" in url

# Create tool objects
search_tool = Tool(
    name="web_search",
    description="Search for information on the web",
    func=web_search
)

verify_tool = Tool(
    name="verify_source",
    description="Check if a source is credible",
    func=verify_source
)

# Create agent with tools
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[search_tool, verify_tool],
    system_message="You are a researcher. Use tools to find and verify information."
)

result = await researcher.run(task="Research sustainable energy")
```

---

## Stage 3: Create Team

```python
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.agents import AssistantAgent

# Create specialized agents
researcher = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[search_tool],
    system_message="Research and find information"
)

analyzer = AssistantAgent(
    name="analyzer",
    model_client=model_client,
    system_message="Analyze data and extract insights"
)

# Create team
team = RoundRobinGroupChat(
    participants=[researcher, analyzer],
    max_turns=10
)

# Run
result = await team.run(task="Research AI trends and analyze impact")
print(f"Completed: {result.stop_reason}")
```

---

## Stage 4: Add Reviewer

```python
from autogen_agentchat.termination import StopMessageTermination

reviewer = AssistantAgent(
    name="reviewer",
    model_client=model_client,
    system_message="""
    Review the analysis. 
    Say "APPROVED" if quality is good.
    Say "REVISION NEEDED" if needs improvement.
    """
)

team = RoundRobinGroupChat(
    participants=[researcher, analyzer, reviewer],
    max_turns=15,
    termination_condition=StopMessageTermination(
        stop_words=["APPROVED", "REVISION NEEDED"]
    )
)

result = await team.run(
    task="Research AI trends, analyze, and get approval"
)

# Check if approved
final_message = result.messages[-1].content
if "APPROVED" in final_message:
    print("✓ Analysis approved for publication")
else:
    print("✗ Needs revision")
```

---

## Stage 5: Human Approval

```python
from autogen_agentchat.agents import UserProxyAgent

human = UserProxyAgent(
    name="human_approver",
    input_func=input
)

team = RoundRobinGroupChat(
    participants=[researcher, analyzer, human],
    max_turns=12
)

result = await team.run(
    task="Complete analysis and get human approval"
)

# Human approved if said so
if any("approved" in msg.content.lower() for msg in result.messages):
    print("Human approved!")
```

---

## Stage 6: State Persistence

```python
import json
from datetime import datetime

async def save_checkpoint(result, filename):
    """Save execution state"""
    checkpoint = {
        "timestamp": datetime.now().isoformat(),
        "messages": [msg.model_dump() for msg in result.messages],
        "stop_reason": result.stop_reason,
        "metadata": {
            "total_messages": len(result.messages),
            "agents": list(set(m.source for m in result.messages))
        }
    }
    
    with open(filename, "w") as f:
        json.dump(checkpoint, f, indent=2)
    
    print(f"Checkpoint saved: {filename}")

# Save after each stage
result = await team.run(task="...")
await save_checkpoint(result, "analysis_checkpoint.json")
```

---

## Stage 7: Observability & Monitoring

```python
import logging
from collections import Counter

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('analysis.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

async def monitored_run(team, task):
    """Run with monitoring"""
    
    logger.info(f"Starting: {task}")
    
    event_count = 0
    async for event in team.run_stream(task=task):
        event_count += 1
        
        if event["type"] == "message_created":
            msg = event["message"]
            logger.info(f"{msg.source}: {msg.content[:100]}...")
        
        elif event["type"] == "team_finished":
            result = event["result"]
            logger.info(f"Completed: {result.stop_reason}")
            
            # Analyze
            speakers = Counter(m.source for m in result.messages)
            logger.info(f"Participation: {dict(speakers)}")
            
            return result
    
    return None

result = await monitored_run(team, "Complete analysis task")
```

---

## Stage 8: Error Handling

```python
from autogen_agentchat.termination import MaxMessageTermination

team = RoundRobinGroupChat(
    participants=[researcher, analyzer, reviewer],
    max_turns=20,
    termination_condition=MaxMessageTermination(max_messages=100)
)

async def run_with_error_handling():
    """Run with error recovery"""
    
    try:
        result = await team.run(task="Complete analysis")
        
        if result.stop_reason == "max_messages":
            logger.warning("Hit message limit, analysis incomplete")
            # Use partial results
            return result.messages[-5:]  # Get last 5 messages
        
        elif result.stop_reason == "error":
            logger.error(f"Error: {result.error}")
            return None
        
        else:
            logger.info("Completed successfully")
            return result.messages
    
    except Exception as e:
        logger.error(f"Exception: {type(e).__name__}: {e}")
        return None

messages = await run_with_error_handling()
```

---

## Complete Workflow

```python
import asyncio
import json
import logging
from datetime import datetime

# Setup
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ResearchWorkflow:
    """Complete research workflow"""
    
    def __init__(self):
        self.model_client = OpenAIClient(
            model="gpt-4-turbo",
            api_key=os.getenv("OPENAI_API_KEY")
        )
        self.checkpoint_file = "research_checkpoint.json"
    
    async def run_complete(self):
        """Run complete workflow"""
        
        # Setup team
        team = RoundRobinGroupChat(
            participants=[
                self.create_researcher(),
                self.create_analyzer(),
                self.create_reviewer()
            ],
            max_turns=15,
            termination_condition=StopMessageTermination(
                stop_words=["APPROVED"]
            )
        )
        
        # Run with monitoring
        logger.info("Starting research workflow")
        result = await self.monitored_run(team, "Research and analyze trends")
        
        # Save checkpoint
        await self.save_checkpoint(result)
        
        # Generate report
        report = self.generate_report(result)
        
        logger.info("Workflow complete")
        return report
    
    def create_researcher(self):
        return AssistantAgent(
            name="researcher",
            model_client=self.model_client,
            tools=[search_tool],
            system_message="Research and find reliable information"
        )
    
    def create_analyzer(self):
        return AssistantAgent(
            name="analyzer",
            model_client=self.model_client,
            system_message="Analyze findings and extract insights"
        )
    
    def create_reviewer(self):
        return AssistantAgent(
            name="reviewer",
            model_client=self.model_client,
            system_message="Review quality and say APPROVED or REVISION NEEDED"
        )
    
    async def monitored_run(self, team, task):
        """Run with monitoring"""
        async for event in team.run_stream(task=task):
            if event["type"] == "team_finished":
                return event["result"]
    
    async def save_checkpoint(self, result):
        """Save state"""
        checkpoint = {
            "timestamp": datetime.now().isoformat(),
            "messages": len(result.messages),
            "stop_reason": result.stop_reason
        }
        with open(self.checkpoint_file, "w") as f:
            json.dump(checkpoint, f)
    
    def generate_report(self, result):
        """Generate report from results"""
        return {
            "status": "complete",
            "messages": len(result.messages),
            "stop_reason": result.stop_reason,
            "final_analysis": result.messages[-1].content if result.messages else ""
        }

# Run workflow
workflow = ResearchWorkflow()
report = await workflow.run_complete()
print(json.dumps(report, indent=2))
```

---

## Key Learnings from This Project

✅ **Team orchestration**: Multiple agents working together  
✅ **Tool integration**: Agents using external functions  
✅ **Termination conditions**: Stopping when goal achieved  
✅ **Human approval**: Including human decision-making  
✅ **Persistence**: Saving work for later  
✅ **Observability**: Monitoring and logging  
✅ **Error handling**: Graceful degradation  

---

## Common Extensions

### Add API Endpoint

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/analyze")
async def analyze(task: str):
    workflow = ResearchWorkflow()
    report = await workflow.run_complete()
    return report
```

### Add Dashboard

```python
# Stream events to WebSocket for live dashboard
async def stream_to_dashboard(task: str):
    async for event in team.run_stream(task=task):
        # Send to WebSocket
        await broadcast(event)
```

### Add Cost Tracking

```python
# Track spending
tokens_used = sum(count_tokens(m.content) for m in result.messages)
cost = (tokens_used / 1000) * 0.01
logger.info(f"Cost: ${cost:.4f}")
```

---

## Next Steps

- Deploy to production
- Add more specialized agents
- Integrate with your data sources
- Set up monitoring dashboard
- Optimize for cost/performance

---

## References

- All previous parts (01-24)
- Official AutoGen: https://microsoft.github.io/autogen/
