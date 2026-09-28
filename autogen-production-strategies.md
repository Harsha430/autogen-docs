# AutoGen Production Strategies: Reliability & Failure Handling

## Moving From Demo to Production

AutoGen demo works:
```
agent.run(task="...") → ✅ Result
```

AutoGen production system:
```
request → validation → retry → fallback → escalation → response
         + logging + monitoring + metrics + tracing
         + error classification + recovery + graceful degradation
```

This document bridges that gap.

---

## Part 1: Failure Taxonomy

### The Stack: Where Failures Occur

```
User Input
    ↓
Orchestration Layer (Team routing, agent selection)
    ↓
Agent Layer (Decision-making, tool selection)
    ↓
LLM Layer (Model inference, token parsing)
    ↓
Tool/External Systems Layer (API calls, database)
    ↓
Storage Layer (State persistence, logging)
    ↓
Response to User
```

Each layer has distinct failure modes.

---

### LLM Failures

**API Failures**
- Provider unavailable (outage)
- Invalid API key / authentication failure
- Quota exceeded / rate limit
- Unauthorized model access
- Network timeout

**Response Failures**
- Malformed response (unparseable JSON)
- Invalid structured output format
- Missing required fields
- Type mismatch in output
- Invalid tool calls in response

**Context Failures**
- Input too long (exceeds context window)
- Token limit exceeded mid-conversation
- Context overflow during streaming
- Invalid prompt formatting

**Behavioral Failures**
- Refusal (model refuses task)
- Incoherent output
- Hallucinated facts
- Inconsistent with prior messages
- Invalid tool parameters generated

**Provider-Specific**
- OpenAI: rate limit, billing issue, model deprecated
- Azure: deployment not found, region unavailable
- Anthropic: overloaded, quota exceeded
- Custom: auth failure, endpoint down

**Recovery Strategy**

| Failure | Transient? | Retry? | Fallback | Escalate |
|---------|-----------|--------|----------|----------|
| Rate limit | Yes | Yes (backoff) | Cheaper model | No |
| Invalid API key | No | No | Check config | Yes |
| Timeout | Maybe | Yes (backoff) | Cached response | Yes |
| Malformed response | No | No (code bug) | N/A | Yes |
| Context overflow | Maybe | No (reduce context) | Summarize | Maybe |
| Model refuses | No | No | Different prompt | Maybe |

---

### Agent Failures

**Logic Failures**
- Wrong tool selected
- Wrong arguments provided
- Repeated tool calls (loop detection)
- Termination not reached
- Invalid handoff

**Behavioral Failures**
- Hallucination (claiming something false)
- Ignoring constraints
- Not following instructions
- Inconsistent decisions
- Infinite reflection loop

**State Failures**
- Agent state corrupted
- State cannot be serialized
- State deserialization fails
- State version incompatible

**Recovery Strategy**

| Failure | Detection | Recovery | Fallback |
|---------|-----------|----------|----------|
| Wrong tool | Trajectory inspection | Re-prompt | Manual override |
| Loop | Message count | Terminate | Simplified workflow |
| Hallucination | Fact-checking | Retry with constraints | Human review |
| State corruption | Load failure | Clear state | Restart fresh |

---

### Tool Failures

**Input Validation**
- Invalid argument type
- Missing required field
- Out-of-range value
- Malformed input format

**Execution Failures**
- Timeout (external API slow)
- Authentication failure (invalid credentials)
- Permission denied (insufficient access)
- External system unavailable
- Network error / unreachable

**Output Failures**
- Malformed response from tool
- Unexpected data format
- Missing required fields
- Corrupted data

**Side-Effect Failures**
- Request succeeded but side effect failed
- Partial completion
- Duplicate execution risk

**Recovery Strategy**

| Failure | Transient? | Retry? | Fallback | Idempotent? |
|---------|-----------|--------|----------|------------|
| Timeout | Yes | Yes (backoff) | Cached | Check API |
| Auth failure | No | No | Refresh credentials | Yes |
| Not found | No | No | Return null | Yes |
| Permission denied | No | No | Escalate | Yes |
| API rate limit | Yes | Yes (exponential backoff) | Cached | Yes |
| Partial failure | Maybe | Maybe (carefully) | Manual intervention | Check logs |

---

### Team/Orchestration Failures

**Routing Failures**
- SelectorGroupChat cannot choose valid agent
- Expected agent unavailable
- HandoffMessage target doesn't exist
- Ambiguous routing decision

**Termination Failures**
- Termination condition never triggered
- max_turns exceeded but work incomplete
- Premature termination
- Deadlock (agents waiting on each other)

**Message Flow Failures**
- Message lost
- Wrong message to wrong agent
- Serialization failure
- Event queue overflow

**Recovery Strategy**

| Failure | Recovery |
|---------|----------|
| Invalid handoff target | Terminate gracefully, log error |
| Selector deadlock | Default to first available agent |
| Premature termination | Retry with increased max_turns |
| Lost message | Re-run with same seed (if deterministic) |

---

### External System Failures

**Database**
- Connection timeout
- Authentication failure
- Database unavailable
- Query error

**APIs / Microservices**
- Service unavailable
- Timeout
- Rate limit
- Invalid response

**Storage (Persistence)**
- Filesystem full
- Permission denied
- File corrupted
- Path invalid

**Recovery Strategy**

| Failure | Transient? | Retry? | Fallback |
|---------|-----------|--------|----------|
| Connection timeout | Yes | Yes (with backoff) | Cache |
| Unavailable | Yes | Yes (with circuit breaker) | Graceful degradation |
| Rate limit | Yes | Yes (exponential backoff) | Queue request |
| Permission error | No | No | Config error / escalate |
| Disk full | No | No | Cleanup / escalate |

---

## Part 2: Fallback Strategies

Fallback ≠ Retry. Fallback means trying a different approach when the primary fails.

### Model Fallback

**Strategy: Degrading Model Tier**

```
GPT-4 (expensive, powerful)
    ↓ failure/timeout
GPT-4-Turbo (balanced)
    ↓ failure/timeout
GPT-3.5-Turbo (cheap, basic)
    ↓ failure
Fallback response / human escalation
```

**When to Use**
- Expensive model timeout
- Provider outage
- Rate limit on primary
- Cost control

**Considerations**
- Output compatibility: Can fallback model produce same format?
- Capability gap: Will quality degrade noticeably?
- Cost trade-off: What's acceptable?
- Latency: Is fallback model faster?

**Implementation**

```python
def get_model_client(tier="primary"):
    models = {
        "primary": OpenAIClient(model="gpt-4"),
        "secondary": OpenAIClient(model="gpt-4-turbo"),
        "tertiary": OpenAIClient(model="gpt-3.5-turbo"),
    }
    return models.get(tier)

async def run_with_model_fallback(task):
    for tier in ["primary", "secondary", "tertiary"]:
        try:
            client = get_model_client(tier)
            result = await agent.run(task=task, model_client=client)
            return result, tier
        except TimeoutError:
            continue
        except Exception as e:
            if tier == "tertiary":
                raise  # Last tier, escalate
            continue
    
    return None, "all_failed"
```

---

### Tool Fallback

**Strategy: Fallback Data Source**

```
Primary: Fresh GitHub API data
    ↓ failure
Secondary: Cached data from 1 hour ago
    ↓ failure
Tertiary: Stale data from yesterday
    ↓ failure
Error: "Data unavailable, try later"
```

**Important**: Clearly indicate data freshness to user.

**When to Use**
- External API unreliable
- Need guaranteed response
- Stale data better than no data

**When NOT to Use**
- Financial data (stale = wrong)
- Time-sensitive queries (weather, stocks)
- Security decisions (permissions)

**Implementation**

```python
async def get_data_with_fallback(query):
    # Tier 1: Fresh data
    try:
        data = await github_api.get_latest(query)
        return data, "fresh"
    except TimeoutError:
        pass
    
    # Tier 2: 1-hour cache
    try:
        data = cache.get_recent(query)  # within 1 hour
        if data:
            return data, "cached_1h"
    except:
        pass
    
    # Tier 3: Old cache
    try:
        data = cache.get_old(query)  # any age
        if data:
            return data, "cached_old"
    except:
        pass
    
    # Tier 4: Failure
    raise DataUnavailableError("All sources failed")
```

---

### Agent Fallback

**Strategy: Backup Specialists**

```
Primary: Research Agent (specialized)
    ↓ failure/timeout/bad output
Secondary: General Assistant (fallback)
    ↓ failure
Tertiary: Template-based response
    ↓ failure
Error: Escalate to human
```

**Implementation**

```python
async def run_with_agent_fallback(task):
    agents = [
        research_agent,
        general_assistant,
        simplified_agent
    ]
    
    for agent in agents:
        try:
            result = await agent.run(task=task)
            if validate_result(result):
                return result, agent.name
        except Exception as e:
            logger.warning(f"{agent.name} failed: {e}")
            continue
    
    # All failed
    logger.error("All agents failed")
    return None, "escalate_human"
```

---

### Routing Fallback

**Problem**: SelectorGroupChat can't decide next agent

**Strategy**

```
Selector tries to choose agent
    ↓
Invalid choice (agent unavailable/doesn't exist)
    ↓
Fallback 1: First available agent
    ↓
Fallback 2: Round-robin
    ↓
Fallback 3: Terminate
```

**Implementation**

```python
async def safe_selector_decision(current_state, available_agents):
    # Try selector
    try:
        chosen = await selector_model.choose(current_state, available_agents)
        if chosen in [a.name for a in available_agents]:
            return chosen
    except:
        pass
    
    # Fallback 1: First available
    if available_agents:
        return available_agents[0].name
    
    # Fallback 2: Terminate
    raise TerminationError("No agents available")
```

---

### Graceful Degradation

**Principle**: System should degrade to simpler versions, not crash.

**Example: Research System**

Full system:
```
Researcher → Tool validator → Analyzer → Reviewer → Writer
```

If tool validator fails:
```
Researcher → Analyzer → Reviewer → Writer
(More risk, but workflow continues)
```

If reviewer fails:
```
Researcher → Analyzer → Writer
(No quality check, but deliverable produced)
```

If analyzer fails:
```
Researcher → Writer
(Minimal processing, raw findings)
```

**Implementation**

```python
async def run_research_with_degradation(task):
    stages = [
        ("researcher", researcher),
        ("validator", validator),
        ("analyzer", analyzer),
        ("reviewer", reviewer),
        ("writer", writer),
    ]
    
    active_stages = stages.copy()
    current_state = task
    
    for name, agent in active_stages:
        try:
            result = await agent.run(task=current_state)
            current_state = result
        except Exception as e:
            logger.warning(f"Stage {name} failed: {e}")
            logger.info(f"Degrading: skipping {name}")
            # Continue with remaining stages
            continue
    
    return current_state
```

---

## Part 3: Retry Strategies

### Retry vs Fallback vs Timeout

**Retry**: Try the same thing again

```
request → failure → retry → failure → retry → success
```

**Fallback**: Try something different

```
request → failure → try alternative → success
```

**Timeout**: Stop trying after time limit

```
request → (wait N seconds) → timeout
```

**Circuit Breaker**: Stop trying after repeated failures

```
request → success ✓
request → failure ✗
request → failure ✗
Circuit OPEN: reject requests for 30s
After 30s: try again
```

---

### Retry Classification

**Transient Failures (Retry)**
- Network timeout
- Rate limit (429)
- Service temporarily unavailable (503)
- Connection refused

**Permanent Failures (Don't Retry)**
- Invalid API key (401)
- Not found (404)
- Bad request (400)
- Permission denied (403)

**Unknown (Carefully Retry)**
- Generic error
- Unexpected response format
- Unknown status code

---

### Backoff Strategies

**Immediate Retry**

```
request → failure → immediate retry
```

**Problem**: If service is temporarily down, hammering with requests makes it worse.

**Fixed Delay**

```
request → failure → wait 2s → retry → failure → wait 2s → retry
```

**Problem**: If service is recovering, 2s might always be too soon.

**Exponential Backoff**

```
request → failure → wait 1s → retry
           failure → wait 2s → retry
           failure → wait 4s → retry
           failure → wait 8s → retry
           failure → wait 16s → max(16s)
```

**Problem**: Predictable pattern. If system has cyclic recovery every 4s, you'll keep hitting it.

**Exponential Backoff + Jitter**

```
wait_time = min(base * 2^attempt, max_wait) + random(0, base * 2^attempt)
```

**Example**

```python
import random

def calculate_backoff(attempt, base=1, max_wait=60):
    exponential = min(base * (2 ** attempt), max_wait)
    jitter = random.uniform(0, exponential)
    return exponential + jitter

# Attempt 0: 0-2 seconds
# Attempt 1: 0-4 seconds
# Attempt 2: 0-8 seconds
# Attempt 3: 0-16 seconds
```

---

### Retry Implementation

```python
async def call_with_retry(
    func,
    max_retries=3,
    base_wait=1,
    max_wait=60,
    retry_on=(TimeoutError, ConnectionError)
):
    """Call func with exponential backoff retry."""
    
    for attempt in range(max_retries + 1):
        try:
            return await func()
        
        except Exception as e:
            # Check if retryable
            if not isinstance(e, retry_on):
                raise  # Not retryable, fail immediately
            
            # Last attempt?
            if attempt == max_retries:
                logger.error(f"Failed after {max_retries} retries: {e}")
                raise
            
            # Calculate backoff
            wait_time = min(base_wait * (2 ** attempt), max_wait)
            jitter = random.uniform(0, wait_time)
            
            logger.warning(
                f"Attempt {attempt + 1} failed: {e}. "
                f"Retrying in {wait_time + jitter:.1f}s"
            )
            
            await asyncio.sleep(wait_time + jitter)
```

---

### When NOT to Retry

**Side Effects**

```python
def create_payment(amount):
    # Request times out AFTER payment was created
    # Retry would create DUPLICATE PAYMENT
    # ❌ BAD
    pass
```

**Solution: Idempotency**

```python
def create_payment(amount, idempotency_key=None):
    # If same idempotency_key: return existing payment
    # If new idempotency_key: create new payment
    # ✅ SAFE
    pass
```

---

## Part 4: Timeout Strategies

### Timeout at Each Layer

| Layer | Typical Timeout | Meaning |
|-------|-----------------|---------|
| Model API call | 30-60s | LLM response |
| Tool execution | 10-30s | External API |
| Agent iteration | 5-10min | One agent turn |
| Team/workflow | 15-30min | Full workflow |

**Important**: Timeouts should be nested.

```
Team timeout (30min)
  ↓
Agent timeout (5min)
  ↓
Model timeout (60s)
  ↓
Tool timeout (30s)
```

If tool times out, agent catches and retries.
If agent times out, team catches and handles.

---

### Timeout + Retry

```
request
  ↓ timeout
retry (attempt 1)
  ↓ timeout
retry (attempt 2)
  ↓ success
```

---

### Timeout + Fallback

```
request
  ↓ timeout
fallback (cache / simpler model)
  ↓
response
```

---

## Part 5: Circuit Breaker

### Problem

```
Service is down
Agent keeps trying
Each request fails
Wasting time and resources
```

### Solution: Circuit Breaker

```
CLOSED (normal operation)
  ↓ success rate > 95%
  ↓
  ↓ success rate drops to < 50%
  ↓
OPEN (stop trying)
  ↓ wait 30s for recovery
  ↓
HALF-OPEN (test 1 request)
  ↓ success? → CLOSED
  ↓ failure? → OPEN (wait longer)
```

### Implementation

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    async def call(self, func):
        """Call func with circuit breaker protection."""
        
        if self.state == "OPEN":
            # Check if recovery period passed
            if (time.time() - self.last_failure_time) > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError("Circuit is OPEN")
        
        try:
            result = await func()
            
            # Success
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
            self.failure_count = 0
            
            return result
        
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            
            raise

# Usage
breaker = CircuitBreaker(failure_threshold=5)

async def call_external_api():
    try:
        return await breaker.call(api.fetch)
    except CircuitBreakerOpenError:
        return None  # Fallback
```

---

## Part 6: Error Classification

### Classification Framework

```
Transient
├─ Network timeout
├─ Rate limit
├─ Service temporarily down
└─ Connection refused

Permanent
├─ Invalid API key
├─ Bad configuration
├─ Invalid input
└─ Permission denied

Business Error
├─ Invalid loan amount
├─ Customer not found
├─ Account locked
└─ Transaction cancelled

Safety Error
├─ Prompt injection
├─ Jailbreak attempt
├─ Policy violation
└─ Dangerous action

Unknown
└─ Unexpected format
```

### Using Classification

| Error Type | Retry? | Fallback | Escalate | Log |
|-----------|--------|----------|----------|-----|
| Transient | Yes | Maybe | No | Warning |
| Permanent | No | Maybe | Yes | Error |
| Business | No | Maybe | Yes | Info |
| Safety | No | No | Yes | Error + Alert |
| Unknown | No | No | Yes | Error |

---

## Part 7: Production Error Pipeline

```
Request
  ↓
Validation
  ├─ Input schema ✓
  ├─ Permissions ✓
  ├─ Rate limit ✓
  ↓
Orchestration
  ├─ Agent selection ✓
  ├─ Tool validation ✓
  ├─ Model client ready ✓
  ↓
Execution
  ├─ Model call
  ├─ Tool call
  ├─ Team coordination
  ↓ ERROR?
  ├─ Classify
  ├─ Retry?
  ├─ Fallback?
  ├─ Escalate?
  ↓
Logging
  ├─ Error type
  ├─ Attempt count
  ├─ Fallback used
  ↓
Metrics
  ├─ Failure count
  ├─ Recovery success
  ├─ Cost
  ↓
Tracing
  ├─ Request ID through chain
  ├─ Blame point
  ├─ Recovery path
  ↓
Response
  ├─ Success with result
  ├─ Partial result
  ├─ Error message
  └─ Status code
```

---

## Part 8: Observability for Failures

### What to Log

**Essential**
```python
log_data = {
    "timestamp": datetime.now().isoformat(),
    "request_id": request_id,  # Trace through system
    "error_type": "TimeoutError",
    "error_message": str(e),
    "error_traceback": traceback.format_exc(),
    "severity": "ERROR",  # ERROR, WARNING, INFO
}
```

**For Retry**
```python
log_data = {
    "attempt": 2,
    "max_attempts": 3,
    "backoff_seconds": 2.5,
    "retry_reason": "Rate limit",
}
```

**For Fallback**
```python
log_data = {
    "fallback_triggered": True,
    "fallback_type": "model",  # model, tool, agent
    "fallback_from": "gpt-4",
    "fallback_to": "gpt-3.5-turbo",
}
```

**For Context**
```python
log_data = {
    "agent": "researcher",
    "tool": "web_search",
    "model": "gpt-4",
    "stage": "tool_execution",
    "tokens_input": 5000,
    "tokens_output": 1000,
}
```

### Structured Logging

```python
import json
import logging

logger = logging.getLogger(__name__)

def log_error(error, context):
    """Log structured error."""
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "level": "ERROR",
        "error_type": type(error).__name__,
        "error_message": str(error),
        **context  # Merge in additional context
    }
    logger.error(json.dumps(log_entry))
```

---

## Part 9: Metrics for Production

### Reliability Metrics

```python
success_rate = successful_requests / total_requests
failure_rate = failed_requests / total_requests
retry_rate = retried_requests / total_requests
fallback_rate = fallback_triggered / total_requests
escalation_rate = human_escalations / total_requests
```

### Performance Metrics

```python
p50_latency = 50th percentile of request latency
p95_latency = 95th percentile
p99_latency = 99th percentile
mean_latency = average

# Per agent
agent_latency = time spent in agent
tool_latency = time spent calling tools
model_latency = time spent in LLM
```

### Cost Metrics

```python
input_tokens_total = sum of input tokens
output_tokens_total = sum of output tokens
total_tokens = input + output
cost = (input_tokens / 1000) * input_rate + (output_tokens / 1000) * output_rate
cost_per_request = total_cost / request_count
```

### Agent Behavior Metrics

```python
tool_calls_per_request = average number of tools called
tool_call_accuracy = correct_tool_calls / total_tool_calls
handoff_accuracy = correct_handoffs / total_handoffs
termination_accuracy = correct_terminations / total_terminations
loop_rate = requests_that_looped / total_requests
```

---

## Part 10: SLO / SLA Concepts

### Definitions

**SLI (Service Level Indicator)**
- Measurable aspect of service
- Example: "request success percentage"

**SLO (Service Level Objective)**
- Target for SLI
- Example: "99% successful requests"

**SLA (Service Level Agreement)**
- Contractual commitment
- Example: "99% uptime or customer gets refund"

### SLO Examples

For agent system:

```
SLI: successful_request_percentage
SLO: >= 98%
Error budget: 2% failures allowed

SLI: p95_latency
SLO: <= 5 seconds
Error budget: 5% of requests can exceed 5s

SLI: cost_per_request
SLO: <= $0.50
Error budget: 5% can exceed
```

### Error Budget

```
Monthly SLO: 99.9% success
Error budget: 0.1% failures
Requests per month: 1,000,000
Allowed failures: 1,000

If 500 failed this week, remaining budget: 500 for month
```

---

## Part 11: Production Checklist

```
RELIABILITY
☐ All models have fallback
☐ All tools have timeout
☐ All tools have retry (if idempotent)
☐ Circuit breaker for flaky services
☐ Retry strategy defined per failure type

FAILURE HANDLING
☐ Error classification implemented
☐ Transient vs permanent identified
☐ Fallback strategies defined
☐ Human escalation defined
☐ Graceful degradation tested

OBSERVABILITY
☐ Structured logging implemented
☐ Error logging captures context
☐ Request tracing (correlation ID)
☐ Metrics collection configured
☐ Dashboards created

TESTING
☐ Failure scenarios tested
☐ Retry tested with backoff
☐ Fallback tested
☐ Escalation tested
☐ State recovery tested

DEPLOYMENT
☐ Gradual rollout plan
☐ Monitoring enabled before deploy
☐ Alerting thresholds set
☐ Rollback plan documented
☐ On-call runbook created

OPERATIONS
☐ Error handling documented
☐ Escalation path clear
☐ SLO defined
☐ SLA defined
☐ Cost budget defined
```

---

## Part 12: Production Scenarios

### Scenario 1: Model Timeout

**Situation**
```
GPT-4 API timeout on 10% of requests
```

**Detection**
```python
if execution_time > timeout_threshold:
    classify_error("timeout", "transient")
```

**Monitoring**
```
Alert: timeout_rate > 5% in 5-minute window
```

**Recovery**
```
1. Immediate: Retry with exponential backoff
2. If persistent: Switch to GPT-4-Turbo (faster)
3. If still failing: Fall back to GPT-3.5-Turbo
4. If all fail: Queue request, respond with "processing"
```

**Log**

```json
{
  "error": "timeout",
  "model": "gpt-4",
  "attempt": 2,
  "timeout_seconds": 60,
  "fallback": "gpt-4-turbo",
  "recovery_success": true
}
```

---

### Scenario 2: Tool Failure (API Down)

**Situation**
```
Web search API returns 503 Service Unavailable
```

**Detection**
```
Tool execution fails with 503
```

**Recovery**
```
1. Immediate: Retry with backoff
2. After 3 failures: Open circuit breaker
3. Fallback: Use cached search results from 1 day ago
4. If no cache: Return "data temporarily unavailable"
```

---

### Scenario 3: Rate Limit

**Situation**
```
OpenAI returns 429 Too Many Requests
```

**Detection**
```
Error code 429 → Classify as transient
```

**Recovery**
```
1. Extract retry-after header
2. Wait retry-after + jitter
3. Retry request
4. If repeated: Reduce request rate (queue requests)
5. If severe: Switch to cheaper model (reduces API calls)
```

---

### Scenario 4: Invalid Tool Arguments

**Situation**
```
Agent generates search_web(query=123)  # int instead of str
Tool call fails validation
```

**Detection**
```
Tool schema validation fails
```

**Recovery**
```
1. NOT retryable (permanent error)
2. Log: "Invalid tool call"
3. Inform agent of error with correction
4. Agent regenerates with correct format
5. If repeated: Escalate to human
```

---

### Scenario 5: Agent Infinite Loop

**Situation**
```
Agent keeps calling same tool
Message count: 100+ and still going
```

**Detection**
```
if message_count > max_messages:
    alert("Infinite loop detected")
```

**Recovery**
```
1. Immediate: Terminate agent
2. Log: Loop detected at message 100
3. Analyze: Why didn't termination condition trigger?
4. Fallback: Return best answer so far
5. Review agent behavior offline
```

---

### Scenario 6: Reviewer Rejects Every Response

**Situation**
```
Reviewer agent always says "REJECTED"
Agent never terminates
Team hits max_turns
```

**Detection**
```
if rejections > N:
    alert("Reviewer always rejecting")
```

**Recovery**
```
1. Terminate workflow (hit max_turns)
2. Fallback: Return result from analyzer (before reviewer)
3. Log: "Reviewer override - using pre-review version"
4. Investigate: Is reviewer prompt too strict?
```

---

### Scenario 7: Database Unavailable

**Situation**
```
State persistence fails
Cannot load previous state
```

**Detection**
```
state_load() raises ConnectionError
```

**Recovery**
```
1. Log: "State load failed"
2. Start fresh (new state)
3. Alert ops to check database
4. Notify user (if state was critical)
5. Retry state load in 10s (circuit breaker)
```

---

### Scenario 8: Cost Explosion

**Situation**
```
Tokens used: 500K (was typical 50K)
Cost: $5 (was typical $0.50)
```

**Detection**
```
if tokens_used > budget * 2:
    alert("Cost spike: " + tokens_used)
```

**Recovery**
```
1. Immediate: Check if legitimate (big task)
2. If yes: Approve but log
3. If no: Investigate what caused spike
4. Possible causes:
   - Agent looping
   - Large context (too much history)
   - Verbose model responses
5. Mitigation: Reduce context, cheaper model, etc.
```

---

### Scenario 9: Handoff to Non-Existent Agent

**Situation**
```
Agent A sends: HandoffMessage(to="specialized_agent")
Team doesn't have "specialized_agent"
```

**Detection**
```
if target_agent not in team.agents:
    raise HandoffError(f"Agent not found: {target_agent}")
```

**Recovery**
```
1. Log: "Invalid handoff target"
2. Fallback: Use default agent (or first available)
3. Continue workflow
4. Alert: Configuration error
```

---

### Scenario 10: Concurrent State Corruption

**Situation**
```
Two requests modify same state simultaneously
State becomes corrupted
Third request loads corrupted state
```

**Detection**
```
Deserialization fails
State validation fails
```

**Recovery**
```
1. Don't retry (state is corrupted)
2. Isolate corrupted state
3. Alert ops
4. Load from backup (if available)
5. Start fresh run
```

---

### Scenario 11: Model Refuses Request

**Situation**
```
Model returns: "I can't help with that request"
Task cannot proceed
```

**Detection**
```
if "I can't" in response or "not appropriate":
    classify_error("refusal", "permanent")
```

**Recovery**
```
1. NOT retryable
2. Fallback: Rephrase request, try again once
3. If still refuses: Escalate to human
4. Log: "Model refused: [reason]"
```

---

### Scenario 12: Tool Returns Unexpected Format

**Situation**
```
Tool should return JSON
Actually returns: "null" (string)
Agent cannot parse
```

**Detection**
```
json.loads(response) raises JSONDecodeError
```

**Recovery**
```
1. Log: "Malformed tool response"
2. NOT retryable (tool's problem)
3. Inform agent of error
4. Fallback: Use default value
5. Or: Ask tool provider to fix
```

---

### Scenario 13: Streaming Interrupted

**Situation**
```
async for event in team.run_stream():
    Network interruption
    Client disconnects
Event stream stops mid-execution
```

**Detection**
```
try:
    async for event in stream:
        ...
except ConnectionError:
    # Handle
```

**Recovery**
```
1. Save state before network loss
2. Client reconnects
3. Resume from saved state (if possible)
4. Or: Restart from beginning
5. Log: "Stream interrupted at event N"
```

---

### Scenario 14: Token Limit Reached

**Situation**
```
Conversation grows
Reaches model's context limit
LLM cannot process
```

**Detection**
```
if token_count > model_context_limit:
    alert("Context overflow")
```

**Recovery**
```
1. Summarize old messages (compress context)
2. Keep only recent messages
3. Continue with reduced context
4. Or: Switch to larger context model
5. Log: "Context overflow, summarized"
```

---

### Scenario 15: Duplicate Side Effects

**Situation**
```
Payment request times out AFTER processing
Retry creates duplicate payment
```

**Detection**
```
Check payment DB for duplicates
```

**Recovery**
```
1. Implement idempotency keys
2. Deduplicate (merge duplicate payments)
3. Refund customer for duplicate
4. Log: "Duplicate payment detected and refunded"
```

---

## Part 13: SLO Worked Example

### System: Multi-Agent Research

**Requirements**
```
• 95% success rate
• p95 latency < 10 seconds
• Cost < $0.50 per request
• 99.5% availability
```

**Metrics**

```
SLI: request_success_rate
SLO: >= 95%
Error budget: 5% failures per month
Monthly requests: 100,000
Allowed failures: 5,000

SLI: p95_latency
SLO: <= 10 seconds
Meaning: 95% of requests finish in ≤ 10s

SLI: cost_per_request
SLO: <= $0.50
Track: (tokens * rate) / requests

SLI: uptime (availability)
SLO: >= 99.5%
Downtime budget: 3.6 hours per month
```

**Alerting**

```
Alert if:
- success_rate < 93% in 5 min window (2% buffer)
- p95_latency > 12s in 5 min window
- cost_per_request > $0.60 (20% buffer)
- uptime drops below 99.5%
```

**Escalation**

```
If SLO breached:
1. Page on-call
2. Investigate cause
3. Implement recovery
4. Review SLO (is it realistic?)
5. Post-mortem
```

---

## Summary

**Production AutoGen = Demo AutoGen + **

- ✅ Failure taxonomy (know what can break)
- ✅ Fallback strategies (have a plan B)
- ✅ Retry logic (handle transient failures)
- ✅ Timeout protection (prevent hangs)
- ✅ Error classification (know how to recover)
- ✅ Observability (see what's happening)
- ✅ Metrics (measure what matters)
- ✅ SLO/SLA (define acceptable)
- ✅ Graceful degradation (service degrades, not crashes)

---

## References

- [15-termination-cancellation.md](15-termination-cancellation.md) - Timeouts and cancellation
- [20-human-in-loop.md](20-human-in-loop.md) - Escalation to humans
- [21-observability.md](21-observability.md) - Logging and monitoring
- [22-performance-cost.md](22-performance-cost.md) - Cost tracking
- [28-troubleshooting.md](28-troubleshooting.md) - Debugging failures
