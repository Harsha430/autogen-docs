# Part 9: Tools - Extending Agent Capabilities

## What Is a Tool?

A **tool** is a Python function that an agent can call to perform actions beyond text generation.

```
Without Tools:
Task → LLM → Response
(LLM can only generate text)

With Tools:
Task → LLM → "Use tool X" → Execute tool → Tool result → LLM → Response
(LLM + function execution)
```

---

## Why Tools Exist

### Problem Without Tools

```python
# Without tools, agent is limited to text
agent = AssistantAgent(
    name="analyst",
    model_client=model_client
)

await agent.run(
    task="Download the latest stock prices and analyze them"
)

# Result: Agent can describe how to get stock prices,
# but cannot actually fetch them
# Output is theoretical, not practical
```

### Solution With Tools

```python
def get_stock_prices(symbols: list[str]) -> dict:
    """Fetch current stock prices"""
    # Actually retrieve data from API
    return {"AAPL": 150.25, "GOOG": 140.30}

agent = AssistantAgent(
    name="analyst",
    model_client=model_client,
    tools=[get_stock_prices]
)

await agent.run(
    task="Download the latest stock prices and analyze them"
)

# Result: Agent actually fetches prices,
# then analyzes them
# Output is practical and actionable
```

---

## Tool System Architecture

```
┌──────────────────────────────────┐
│ AssistantAgent                   │
│                                  │
│ 1. Sees available tools          │
│ 2. Sends tool schemas to LLM     │
│ 3. Receives tool_call from LLM   │
└────────────┬─────────────────────┘
             │
┌────────────▼─────────────────────┐
│ Tool Schema (sent to LLM)        │
│                                  │
│ {                                │
│   "name": "get_stock_prices",   │
│   "description": "...",         │
│   "parameters": {               │
│     "type": "object",           │
│     "properties": {...}         │
│   }                              │
│ }                                │
└────────────┬─────────────────────┘
             │
┌────────────▼─────────────────────┐
│ LLM Processing                   │
│                                  │
│ Sees:                            │
│ - Task                           │
│ - Available tools                │
│ - Tool parameters & descriptions │
│                                  │
│ Decides: Should I call this tool?│
└────────────┬─────────────────────┘
             │
         YES │
             │
┌────────────▼─────────────────────┐
│ LLM Generates Tool Call          │
│                                  │
│ {                                │
│   "name": "get_stock_prices",   │
│   "arguments": {                │
│     "symbols": ["AAPL", "GOOG"] │
│   }                              │
│ }                                │
└────────────┬─────────────────────┘
             │
┌────────────▼─────────────────────┐
│ Agent Executes Tool              │
│                                  │
│ get_stock_prices(               │
│   symbols=["AAPL", "GOOG"]     │
│ )                                │
│                                  │
│ Returns: {"AAPL": 150.25, ...}  │
└────────────┬─────────────────────┘
             │
┌────────────▼─────────────────────┐
│ Add Tool Result to Context       │
│                                  │
│ LLM now sees:                    │
│ - Original task                  │
│ - Tool request                   │
│ - Tool result                    │
└────────────┬─────────────────────┘
             │
┌────────────▼─────────────────────┐
│ LLM Continues Processing         │
│                                  │
│ Analyzes tool result             │
│ Generates final response         │
└─────────────────────────────────┘
```

---

## Creating Tools: The Complete Guide

### Rule 1: Tools Must Be Python Functions

```python
# CORRECT - Function with type hints
def web_search(query: str) -> str:
    """Search the web for information"""
    # Implementation
    return "Search results..."

# CORRECT - Function with multiple parameters
def calculate_compound_interest(
    principal: float,
    rate: float,
    years: int
) -> float:
    """Calculate compound interest"""
    return principal * (1 + rate) ** years

# WRONG - Not a function
web_search = "I search the web"  # String, not function

# WRONG - No type hints
def search(query):  # Missing type hints!
    return "results"
```

### Rule 2: Must Have Type Hints

Type hints are **required**. LLM uses them to understand parameters.

```python
# CORRECT - Clear type hints
from typing import Annotated

def send_email(
    to: Annotated[str, "Email recipient"],
    subject: Annotated[str, "Email subject"],
    body: Annotated[str, "Email body text"]
) -> str:
    """Send an email"""
    # Implementation
    return f"Email sent to {to}"

# WRONG - Missing type hints
def send_email(to, subject, body):
    return f"Email sent to {to}"

# WRONG - Missing return type
def get_data(query: str):  # No return type!
    return "data"
```

### Rule 3: Include Docstring

Docstring explains to LLM what the tool does.

```python
def web_search(query: str) -> str:
    """Search the web for information using Google Search API
    
    This tool searches the internet for current information.
    Use it when you need to find recent news, data, or facts.
    """
    # Implementation
    return "results"

# LLM sees this docstring and understands:
# - What the tool does
# - When to use it
# - What it returns
```

### Rule 4: Include Parameter Descriptions

Use `Annotated` to describe parameters.

```python
from typing import Annotated

def calculate_loan_payment(
    principal: Annotated[float, "Loan amount in dollars"],
    annual_rate: Annotated[float, "Annual interest rate (0.05 for 5%)"],
    months: Annotated[int, "Loan duration in months"]
) -> str:
    """Calculate monthly loan payment"""
    monthly_rate = annual_rate / 12
    payment = principal * (monthly_rate * (1 + monthly_rate) ** months) / \
              ((1 + monthly_rate) ** months - 1)
    return f"Monthly payment: ${payment:.2f}"

# LLM understands each parameter's purpose and format
```

---

## Tool Examples by Category

### Category 1: Data Retrieval

```python
def search_knowledge_base(
    query: Annotated[str, "Search query"],
    max_results: Annotated[int, "Maximum results to return"] = 5
) -> str:
    """Search internal knowledge base"""
    # Query database or search service
    results = db.search(query, limit=max_results)
    return str(results)

def get_current_weather(
    location: Annotated[str, "City and state (e.g., 'New York, NY')"]
) -> str:
    """Get current weather for a location"""
    # Call weather API
    weather = weather_api.get(location)
    return f"Weather in {location}: {weather}"

def fetch_stock_price(
    symbol: Annotated[str, "Stock ticker symbol (e.g., 'AAPL')"]
) -> str:
    """Get current stock price"""
    # Call financial API
    price = finance_api.get_price(symbol)
    return f"{symbol}: ${price}"
```

### Category 2: Computation/Analysis

```python
def analyze_sentiment(
    text: Annotated[str, "Text to analyze"]
) -> str:
    """Analyze sentiment of text"""
    # Use NLP model
    score = sentiment_model.predict(text)
    return f"Sentiment score: {score} (1=positive, -1=negative)"

def summarize_document(
    document: Annotated[str, "Document text to summarize"],
    max_length: Annotated[int, "Maximum length of summary in words"] = 100
) -> str:
    """Summarize a document"""
    # Use summarization model
    summary = summarize(document, max_length)
    return summary

def translate_text(
    text: Annotated[str, "Text to translate"],
    target_language: Annotated[str, "Target language (e.g., 'Spanish', 'French')"]
) -> str:
    """Translate text to another language"""
    # Use translation service
    translated = translator.translate(text, target_language)
    return translated
```

### Category 3: Data Modification

```python
def create_task(
    title: Annotated[str, "Task title"],
    description: Annotated[str, "Task description"],
    priority: Annotated[str, "Priority: 'high', 'medium', or 'low'"] = "medium"
) -> str:
    """Create a new task"""
    task_id = task_manager.create(title, description, priority)
    return f"Task created with ID: {task_id}"

def update_database_record(
    record_id: Annotated[int, "ID of record to update"],
    changes: Annotated[dict, "Changes to make (field: new_value)"]
) -> str:
    """Update a database record"""
    db.update(record_id, changes)
    return f"Record {record_id} updated successfully"

def send_notification(
    recipient: Annotated[str, "Email or user ID"],
    message: Annotated[str, "Notification message"]
) -> str:
    """Send a notification to a user"""
    notify.send(recipient, message)
    return f"Notification sent to {recipient}"
```

### Category 4: Code Execution

```python
def execute_python(
    code: Annotated[str, "Python code to execute"]
) -> str:
    """Execute Python code safely"""
    try:
        result = eval(code)  # In practice, use safer sandbox
        return str(result)
    except Exception as e:
        return f"Error: {e}"

def run_sql_query(
    query: Annotated[str, "SQL SELECT query"]
) -> str:
    """Execute SQL query against database"""
    try:
        results = db.execute(query)
        return str(results)
    except Exception as e:
        return f"Error: {e}"

def test_api_endpoint(
    url: Annotated[str, "API endpoint URL"],
    method: Annotated[str, "HTTP method: 'GET', 'POST', etc"] = "GET"
) -> str:
    """Test an API endpoint"""
    try:
        response = requests.request(method, url)
        return f"Status: {response.status_code}, Body: {response.text[:500]}"
    except Exception as e:
        return f"Error: {e}"
```

---

## Complete Tool Example

```python
from typing import Annotated
import httpx

def web_search(
    query: Annotated[str, "Search query string"],
    num_results: Annotated[int, "Number of results to return (1-10)"] = 5
) -> str:
    """Search the web for information
    
    This tool performs a web search and returns the top results.
    Use this when you need current information, news, or facts.
    
    Args:
        query: What to search for
        num_results: How many results to return (default 5)
    
    Returns:
        String containing search results
    """
    try:
        # In practice, use actual search API (Google, Bing, etc)
        # This is a simplified example
        search_url = f"https://api.example.com/search"
        
        params = {
            "q": query,
            "num": min(num_results, 10)  # Cap at 10
        }
        
        response = httpx.get(search_url, params=params)
        response.raise_for_status()
        
        results = response.json()
        formatted = "\n".join([
            f"{i+1}. {r['title']}\n   {r['url']}\n   {r['snippet']}"
            for i, r in enumerate(results[:num_results])
        ])
        
        return formatted
    
    except Exception as e:
        return f"Search failed: {str(e)}"
```

---

## Passing Tools to Agents

```python
from autogen_agentchat.agents import AssistantAgent

# Define tools
def web_search(query: str) -> str:
    """Search the web"""
    ...

def calculate(expression: str) -> str:
    """Calculate mathematical expression"""
    ...

# Pass as list
agent = AssistantAgent(
    name="analyst",
    model_client=model_client,
    tools=[web_search, calculate],  # List of functions
    system_message="You are an analyst with web and calculation abilities"
)

# Agent will automatically:
# 1. Extract function signatures
# 2. Create JSON schemas
# 3. Send to LLM
# 4. Execute when LLM requests
```

---

## Tool Return Values: Important

Tools return **strings** only (the LLM sees strings).

```python
# CORRECT - Return string
def get_data() -> str:
    data = fetch_from_api()
    return str(data)  # Convert to string

# WRONG - Returns dict
def get_data() -> dict:
    return fetch_from_api()  # LLM can't use dict directly

# CORRECT - Return formatted string
def get_users(limit: int) -> str:
    users = db.get_users(limit)
    return "\n".join([f"- {u.name} (ID: {u.id})" for u in users])

# WRONG - Returns list
def get_users(limit: int) -> list:
    return db.get_users(limit)  # LLM sees list
```

---

## Error Handling in Tools

Tools should handle errors gracefully:

```python
def divide(
    a: Annotated[float, "Numerator"],
    b: Annotated[float, "Denominator"]
) -> str:
    """Divide a by b"""
    try:
        if b == 0:
            return "Error: Cannot divide by zero"
        result = a / b
        return f"{a} / {b} = {result}"
    except Exception as e:
        return f"Calculation error: {str(e)}"

def send_email(
    to: Annotated[str, "Recipient email"],
    subject: Annotated[str, "Email subject"],
    body: Annotated[str, "Email body"]
) -> str:
    """Send an email"""
    try:
        # Validate email format
        if "@" not in to:
            return f"Error: Invalid email address '{to}'"
        
        # Send email
        email_service.send(to, subject, body)
        return f"Email sent successfully to {to}"
    
    except Exception as e:
        return f"Failed to send email: {str(e)}"
```

---

## Multiple Tools

Agents can use multiple tools:

```python
def web_search(query: str) -> str:
    """Search the web"""
    ...

def read_file(path: str) -> str:
    """Read a file"""
    ...

def run_code(code: str) -> str:
    """Execute Python code"""
    ...

agent = AssistantAgent(
    name="researcher",
    model_client=model_client,
    tools=[web_search, read_file, run_code],
    max_tool_iterations=10  # Can chain multiple tools
)

# LLM decides which tools to use and in what order
# Example: Search → Read results → Execute analysis → Return findings
```

---

## Tool Execution Order

LLM decides tool order based on task:

```
Task: "Search for Python news, analyze if positive or negative, 
       send summary email"

LLM's reasoning:
1. First, use web_search to get news
2. Then, use analyze_sentiment on news
3. Finally, use send_email with summary

Agent executes:
1. web_search() → returns news articles
2. analyze_sentiment() → returns positive/negative scores
3. send_email() → sends summary

Each tool result feeds into next LLM call
```

---

## Common Tool Mistakes

### Mistake 1: Missing Type Hints

```python
# WRONG - LLM can't understand parameters
def search(query):  # No type hints!
    return "results"

# FIX
def search(query: str) -> str:
    """Search for information"""
    return "results"
```

### Mistake 2: Returning Non-String

```python
# WRONG - LLM sees Python dict, not readable text
def get_users() -> dict:
    return {"users": [{"id": 1, "name": "Alice"}]}

# FIX - Return formatted string
def get_users() -> str:
    users = [{"id": 1, "name": "Alice"}]
    return "\n".join([f"- {u['name']} (ID: {u['id']})" for u in users])
```

### Mistake 3: Tool Does Too Much

```python
# WRONG - Tool tries to do everything
def process_and_analyze_and_summarize(data: str) -> str:
    """Process, analyze, and summarize"""
    # 300 lines of complex logic
    ...

# FIX - Break into smaller tools
def process_data(data: str) -> str:
    """Process raw data"""
    ...

def analyze_data(data: str) -> str:
    """Analyze processed data"""
    ...

def summarize_results(results: str) -> str:
    """Summarize analysis results"""
    ...

# LLM can chain these smaller tools
```

### Mistake 4: No Documentation

```python
# WRONG - No docstring
def x(y):
    return z

# LLM has no idea what this does

# FIX - Clear documentation
def calculate_total_cost(
    item_price: Annotated[float, "Price per item"],
    quantity: Annotated[int, "Number of items"]
) -> str:
    """Calculate total cost of items
    
    Use this to compute final cost including quantity.
    """
    total = item_price * quantity
    return f"Total cost: ${total:.2f}"
```

---

## Next Steps

Now that you understand tools:
- ✅ What tools are
- ✅ How to create them
- ✅ How to pass them to agents
- ✅ Common mistakes

Move to:
- **[10-single-agent-loop.md](10-single-agent-loop.md)** to see tools in action
- **[04-assistant-agent.md](04-assistant-agent.md)** to review agent parameters again

---

## References

- [AutoGen Tools Documentation](https://microsoft.github.io/autogen/)
- [Python Type Hints](https://docs.python.org/3/library/typing.html)
- [Python Typing - Annotated](https://docs.python.org/3/library/typing.html#typing.Annotated)

---

## Version Notes

- **Current Version**: AutoGen 0.2.x
- **Type Hints**: Required for tool discovery
- **String Returns**: Tools must return strings for LLM processing
