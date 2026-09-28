# AutoGen Evaluation, Metrics & Interview Preparation

## Part A: Evaluating Agent Systems

---

## The Evaluation Problem

**"It works"** ≠ **"It works well"**

```python
result = await agent.run(task="...")
print(result.messages[-1].content)
# Output is plausible
# But is it correct?
```

This document teaches evaluation rigor.

---

## Evaluation Framework

An agent system should be measured across:

```
Correctness
  ↓
Reliability
  ↓
Efficiency
  ↓
Safety
  ↓
Cost
  ↓
Latency
  ↓
User Satisfaction
```

Only one metric is insufficient.

---

## Evaluation Levels

### Level 1: LLM Output Quality

**What**: Does the model's response make sense?

**Metrics**:
- Correctness (is answer right?)
- Relevance (does it address the query?)
- Coherence (is it well-structured?)
- Factuality (are claims true?)

**Example**

```
Task: "What is the capital of France?"

Output: "The capital of France is Paris."

Evaluation:
- Correctness: ✅ Yes
- Relevance: ✅ Directly answers question
- Coherence: ✅ Clear sentence
- Factuality: ✅ Factually correct
```

---

### Level 2: Agent Behavior

**What**: Did the agent make good decisions?

**Metrics**:
- Tool selection accuracy (correct tool?)
- Tool call validity (valid arguments?)
- Stopping criteria (right time to stop?)
- Instruction adherence (followed system message?)

**Example**

```
Task: "Research Python 3.13"

Agent behavior:
1. Chose search_web tool ✅ (correct)
2. Called search_web(query="Python 3.13") ✅ (valid args)
3. Received results
4. Chose to stop ✅ (appropriate)

Evaluation:
- Tool selection: ✅ Correct
- Arguments: ✅ Valid
- Stopping: ✅ Appropriate
```

---

### Level 3: Tool Usage

**What**: Did agents use tools correctly?

**Metrics**:
- Tool call success rate (% successful)
- Tool argument correctness (% valid arguments)
- Tool necessity (called when needed?)
- Tool redundancy (unnecessary calls?)

**Example**

```
10 total tool calls:
- 8 succeeded
- 2 failed (invalid arguments)

Metrics:
- Success rate: 80%
- Argument correctness: 80%
- Necessary: 9/10 were necessary
- Redundancy: 1/10 was redundant
```

---

### Level 4: Agent Trajectory

**What**: Was the sequence of steps logical?

**Metrics**:
- Trajectory correctness (right sequence?)
- Unnecessary steps (wasted effort?)
- Loop detection (repeated steps?)
- Tool selection per-step (right tool at each step?)

**Example**

```
Trajectory:
1. Agent searched for data ✅
2. Agent analyzed results ✅
3. Agent searched AGAIN ❌ (redundant)
4. Agent stopped ✅

Evaluation:
- Step 1-2: Correct sequence
- Step 3: Unnecessary
- No loops detected
- Score: 3/4 steps optimal
```

---

### Level 5: Multi-Agent Behavior

**What**: Did multiple agents coordinate well?

**Metrics**:
- Agent selection accuracy (chose right agent?)
- Handoff correctness (handed off to right agent?)
- Collaboration efficiency (wasted coordination?)
- Deadlock detection (got stuck?)

**Example**

```
Team: Researcher → Analyzer → Writer

Trajectory:
1. Researcher researched ✅
2. Analyzer analyzed ✅
3. Writer wrote ✅
4. Researcher re-researched ❌ (not needed)
5. Analyzer re-analyzed ✅ (needed for new data)
6. Writer finalized ✅

Evaluation:
- Handoffs: 5/5 to correct agents
- Efficiency: 1 unnecessary step
- Deadlock: None
- Score: 4/5 optimal
```

---

### Level 6: End-to-End System

**What**: Did the full system produce correct output?

**Metrics**:
- Task success rate (% tasks succeeded?)
- Output quality (quality of final answer?)
- Time to completion (how long?)
- Tokens used (how much?)

**Example**

```
100 test tasks
- 95 succeeded
- 5 failed

Success rate: 95%
Avg quality score: 8.2/10
Avg latency: 4.5s
Avg tokens: 3,500
```

---

### Level 7: Production Behavior

**What**: Does it work reliably in production?

**Metrics**:
- Uptime (% available?)
- Error rate (% failed?)
- SLO compliance (meets targets?)
- Cost accuracy (within budget?)

**Example**

```
30 days of production:
- 99.8% uptime
- 1.2% error rate
- 98.5% met SLO (97.5% target)
- Cost: $12,000 (budget: $15,000)
```

---

## Output Quality Evaluation

### Correctness

**Definition**: Is the answer factually correct?

**Example**

```
Q: "What is 2+2?"
A: "4"
→ Correct ✅

Q: "What is 2+2?"
A: "5"
→ Incorrect ❌
```

### Relevance

**Definition**: Does the answer address the query?

**Example**

```
Q: "What is the capital of France?"
A: "Paris is a beautiful city in France."
→ Relevant ✅

Q: "What is the capital of France?"
A: "France is in Europe."
→ Not relevant ❌
```

### Completeness

**Definition**: Are all required pieces included?

**Example**

```
Q: "List the top 3 Python frameworks"
A: "Django, Flask"
→ Incomplete (only 2) ❌

Q: "List the top 3 Python frameworks"
A: "Django, Flask, FastAPI"
→ Complete ✅
```

### Factuality & Groundedness

**Definition**: Are claims supported by evidence?

**Example**

```
Q: "Who won the 2024 Nobel Prize in Physics?"
A: "According to the Nobel Prize website, John Smith won."
→ Grounded (cited source) ✅

Q: "Who won the 2024 Nobel Prize in Physics?"
A: "Probably John Smith."
→ Speculative, not grounded ❌
```

---

## Evaluation Metrics Reference

### Tool-Related Metrics

```
tool_selection_accuracy = 
  correct_tool_selections / total_tool_calls

tool_argument_accuracy = 
  valid_tool_arguments / total_tool_calls

tool_call_success_rate = 
  successful_tool_calls / total_tool_calls

tool_call_redundancy = 
  repeated_identical_calls / total_tool_calls
```

### Agent Behavior Metrics

```
instruction_adherence_rate =
  times_followed_instructions / total_tasks

handoff_accuracy =
  correct_handoffs / total_handoffs

termination_accuracy =
  correct_terminations / total_tasks

loop_detection_rate =
  tasks_with_loops / total_tasks
```

### Trajectory Metrics

```
trajectory_optimality =
  optimal_steps / total_steps

unnecessary_steps =
  wasted_steps / total_steps

recovery_success_rate =
  successful_recoveries / total_errors
```

### System-Level Metrics

```
task_success_rate =
  successful_tasks / total_tasks

mean_tokens_per_task =
  total_tokens / total_tasks

mean_latency_per_task =
  total_time / total_tasks

cost_per_successful_task =
  total_cost / successful_tasks
```

---

## Evaluation Dataset

### Building a Test Dataset

Each test case should include:

```python
test_case = {
    "id": "test_001",
    "input": "User query here",
    "expected_output": "What should agent produce?",
    "allowed_tools": ["tool_a", "tool_b"],
    "expected_trajectory": [
        {"step": 1, "tool": "tool_a"},
        {"step": 2, "tool": "tool_b"}
    ],
    "constraints": ["Must not call tool_c"],
    "ground_truth": "Actual correct answer",
    "difficulty": "easy",  # easy, medium, hard
    "category": "research"
}
```

### Golden Dataset Composition

```
Edge cases          10%
  ├─ Empty input
  ├─ Malformed input
  └─ Ambiguous queries

Adversarial        10%
  ├─ Trick questions
  ├─ Contradiction
  └─ Red herrings

Common tasks       50%
  ├─ Research queries
  ├─ Analysis
  └─ Summarization

Regression cases   20%
  ├─ Previously failing cases
  └─ Recently fixed bugs

Safety cases       10%
  ├─ Inappropriate requests
  ├─ Harmful content
  └─ Policy violations
```

---

## LLM-as-Judge Evaluation

### Concept

```
Model Output
    ↓
Judge LLM (separate model)
    ↓
Score + Explanation
```

### Rubric Example

```json
{
  "criteria": [
    {
      "name": "Correctness",
      "scale": 1-5,
      "description": "Is the answer factually correct?"
    },
    {
      "name": "Completeness",
      "scale": 1-5,
      "description": "Does it address all parts of the query?"
    },
    {
      "name": "Clarity",
      "scale": 1-5,
      "description": "Is it well-written and easy to understand?"
    }
  ]
}
```

### Judge Prompt

```
Task: Rate the following output

Output:
[Agent's response]

Criteria:
1. Correctness (1-5): Is it factually correct?
2. Completeness (1-5): Does it fully address the query?
3. Clarity (1-5): Is it well-explained?

Provide:
- Individual scores
- Brief justification
- Overall assessment
```

### Limitations

- **Judge bias**: LLM judges may have biases
- **Correlated errors**: Judge and agent might both fail on same input
- **Inconsistency**: LLM may score inconsistently
- **Contamination**: Judge might recognize test set

---

## Multi-Agent Evaluation

### Metrics for Teams

```
Agent Selection Accuracy:
  Correct next agent chosen / total selections

Handoff Success Rate:
  Successful handoffs / total handoffs

Collaboration Efficiency:
  Unnecessary handoffs / total handoffs

Deadlock Rate:
  Tasks that deadlock / total tasks

Final Output Quality:
  (Evaluated at system level)
```

### Example: 3-Agent Research Team

```
100 research tasks

Agent Selection:
- Researcher chosen 95/95 times at start ✅
- Analyzer chosen 90/95 times after research ✅
- Writer chosen 92/95 times after analysis ✅
- Accuracy: 277/285 = 97.2%

Handoffs:
- Researcher → Analyzer: 93/95 correct
- Analyzer → Writer: 92/95 correct
- Success rate: 185/190 = 97.4%

Deadlock:
- Tasks stuck in loop: 3/100
- Deadlock rate: 3%

Quality:
- Final output quality: 8.5/10 average
```

---

## RAG-Specific Evaluation

### Context Retrieval Metrics

```
context_precision =
  relevant_documents_retrieved / documents_retrieved

context_recall =
  relevant_documents_retrieved / total_relevant_documents

context_f1_score =
  2 * (precision * recall) / (precision + recall)
```

### Answer Quality vs Context

```
Faithfulness =
  claims_supported_by_context / total_claims

Answer_Relevancy =
  relevant_parts_of_answer / total_answer_length
```

### Example

```
Query: "What is AutoGen?"

Retrieved contexts: 5 documents
- 4 are about AutoGen ✅
- 1 is about LangGraph ❌

context_precision = 4/5 = 80%

Total relevant documents in corpus: 8
context_recall = 4/8 = 50%

Generated answer:
"AutoGen is an agent framework by Microsoft."
- Claim: "by Microsoft" → Supported by context ✅
- Claim: "agent framework" → Supported ✅

faithfulness = 2/2 = 100%
```

---

## Regression Testing

### Why Regression Testing for Agents?

Agents don't have fixed behavior like functions.

```
Version 1:
def analyze(data):
    return "positive"  # Fixed

Agent 1:
async def analyze(data):
    # Uses LLM + tools
    # Different run = different result
    # Prompt change? Different behavior
    # Model change? Different behavior
```

### Regression Test Pipeline

```
Baseline Version
    ↓
Run golden dataset (100 tests)
    ↓
Record metrics:
  - success_rate: 95%
  - avg_quality: 8.2/10
  - avg_latency: 4.5s
    ↓
New Version
    ↓
Run same 100 tests
    ↓
Compare metrics:
  - success_rate: 94% (1% regression) ⚠️
  - avg_quality: 8.1/10 (slight regression)
  - avg_latency: 4.6s (1% slower)
    ↓
Decision: Accept (within tolerance) or reject
```

### Regression Thresholds

```
If success_rate drops > 2%: FAIL
If quality drops > 0.5 points: FAIL
If latency increases > 10%: WARN
If cost increases > 20%: WARN
```

---

## Evaluation Report Template

```markdown
# Evaluation Report: Research Agent v1.2

## Test Set
- Total tests: 100
- Categories:
  - Research: 40
  - Analysis: 30
  - Writing: 20
  - Edge cases: 10

## Results

### Success Metrics
- Task success rate: 95%
- Tool selection accuracy: 92%
- Handoff accuracy: 98%
- Termination accuracy: 94%

### Quality Metrics
- Average quality score: 8.2/10
- Correctness: 96%
- Completeness: 90%
- Factuality: 94%

### Performance Metrics
- Mean latency: 4.5s
- p95 latency: 8.2s
- p99 latency: 12.1s

### Cost Metrics
- Mean tokens: 3,500
- Mean cost: $0.35
- Cost distribution: p50=$0.25, p95=$0.65

### Reliability
- Tool failure rate: 3%
- Fallback usage rate: 2%
- Error recovery rate: 87%

## Failures
- 5 tasks failed (5%)
  - 3: Tool timeout
  - 2: Agent loop
- Root cause: Loose termination condition

## Recommendations
1. Tighten termination condition
2. Add timeout to web search tool
3. Test with larger dataset

## Comparison to v1.1
- Success rate: 95% vs 92% (+3%) ✅
- Quality: 8.2/10 vs 7.9/10 (+0.3) ✅
- Latency: 4.5s vs 4.2s (-7%) ⚠️
- Cost: $0.35 vs $0.32 (-9%) ⚠️

## Conclusion
v1.2 is approved for production with improvements to success rate and quality despite slight latency/cost increase.
```

---

# Part B: Interview Preparation

## Level 1: Fundamentals (30 Questions)

### 1. Core Concepts

**Q1: What is AutoGen?**
- **Testing**: Basic understanding
- **Answer**: Agent framework by Microsoft for building multi-agent systems with LLM orchestration
- **Keywords**: Multi-agent, orchestration, LLM, open-source
- **Follow-up**: How does it differ from just using an LLM?

**Q2: What's the difference between an agent and an LLM?**
- **Answer**: 
  - LLM: Language model, generates text
  - Agent: Wrapper around LLM that can: perceive, reason, select tools, take actions
- **Keywords**: Perception, reasoning, tool use, autonomy

**Q3: Why use agents instead of just chaining LLM calls?**
- **Answer**: Agents can make decisions, use tools, adapt to errors. Chains are fixed.
- **Keywords**: Autonomy, adaptability, decision-making

**Q4: What is a Team in AutoGen?**
- **Answer**: Multiple agents working together, coordinated by orchestration layer
- **Keywords**: Multi-agent, coordination, orchestration

**Q5: What is the difference between agent and workflow?**
- **Answer**:
  - Agent: Autonomous, makes decisions
  - Workflow: Fixed sequence of steps
- **Keywords**: Autonomy, flexibility, sequencing

**Q6: What layers does AutoGen have?**
- **Answer**: User layer, orchestration layer (teams), agent layer, LLM layer
- **Keywords**: Abstraction, layering, separation of concerns

**Q7: What is AssistantAgent?**
- **Answer**: Primary agent type: LLM-based, can use tools, has system message
- **Keywords**: LLM-backed, tool-capable, configurable

**Q8: What is UserProxyAgent?**
- **Answer**: Represents human in team; blocks for human input
- **Keywords**: Human-in-loop, blocking, interactive

**Q9: What is a model_client?**
- **Answer**: Abstraction layer for LLM providers (OpenAI, Azure, Anthropic, etc.)
- **Keywords**: Provider agnostic, abstraction, pluggable

**Q10: What are tools?**
- **Answer**: Functions agents can call; defined with schema
- **Keywords**: Function calling, schema, capability extension

### 2. Teams

**Q11: What is RoundRobinGroupChat?**
- **Answer**: Team with fixed speaker order: A→B→C→A→...
- **Keywords**: Fixed order, cyclic, deterministic

**Q12: What is SelectorGroupChat?**
- **Answer**: Team where AI model decides next speaker based on context
- **Keywords**: Dynamic routing, AI-decided, adaptive

**Q13: What is Swarm?**
- **Answer**: Team where agents can handoff to each other autonomously
- **Keywords**: Autonomous handoff, agent-decided, distributed

**Q14: When would you use Swarm instead of SelectorGroupChat?**
- **Answer**: When agents should decide when to handoff (e.g., escalation)
- **Keywords**: Agent autonomy, natural handoff, escalation

**Q15: What is max_turns?**
- **Answer**: Maximum number of agent turns before stopping
- **Keywords**: Termination, limit, safety

### 3. Execution

**Q16: What does run() return?**
- **Answer**: TaskResult with messages, stop_reason
- **Keywords**: TaskResult, final result, blocking

**Q17: What does run_stream() return?**
- **Answer**: Async iterator of events (messages, tool calls, etc.)
- **Keywords**: Streaming, async, events, real-time

**Q18: What is TaskResult?**
- **Answer**: Object containing: messages (list), stop_reason (why it stopped)
- **Keywords**: Final state, messages, completion reason

**Q19: What does stop_reason mean?**
- **Answer**: Why execution stopped (max_turns, termination_condition, error)
- **Keywords**: Termination, completion, status

**Q20: What is a termination_condition?**
- **Answer**: Logic to determine when team should stop
- **Keywords**: Stopping condition, termination, control

### 4. Messages & State

**Q21: What message types exist?**
- **Answer**: TextMessage, HandoffMessage, ToolCallSummaryMessage
- **Keywords**: Communication types, message hierarchy

**Q22: What is a HandoffMessage?**
- **Answer**: Message from agent requesting control transfer to another agent
- **Keywords**: Agent communication, routing signal

**Q23: What is state vs message history?**
- **Answer**:
  - State: Current snapshot (turn count, metadata)
  - History: All messages generated
- **Keywords**: Snapshot, history, temporal

**Q24: What is model_context?**
- **Answer**: Tokens sent to LLM (all messages, conversation history)
- **Keywords**: Context window, token limit, input

**Q25: What is memory?**
- **Answer**: Long-term facts persisted across conversations
- **Keywords**: Persistence, facts, multi-turn

### 5. Failure & Recovery

**Q26: What's the difference between retry and fallback?**
- **Answer**:
  - Retry: Try same thing again
  - Fallback: Try something different
- **Keywords**: Recovery, resilience

**Q27: When should you retry?**
- **Answer**: Transient failures (timeout, rate limit)
- **Keywords**: Transient, idempotent

**Q28: When should you NOT retry?**
- **Answer**: Permanent failures (invalid API key, bad input)
- **Keywords**: Permanent, configuration

**Q29: What is exponential backoff?**
- **Answer**: Wait 1s, 2s, 4s, 8s between retries (+ jitter)
- **Keywords**: Backoff, jitter, spacing

**Q30: What is graceful degradation?**
- **Answer**: System reduces functionality but continues operating
- **Keywords**: Resilience, degradation, continuity

---

## Level 2: API & Implementation (30 Questions)

### AssistantAgent Deep Dive

**Q31: What parameters does AssistantAgent accept?**
- **Answer**: name, model_client, system_message, tools, max_tool_iterations, ...
- **Keywords**: Configuration, constructor, parameters

**Q32: What does max_tool_iterations control?**
- **Answer**: Maximum number of times agent can call tools in one turn
- **Keywords**: Tool loop prevention, iteration limit

**Q33: What does reflect_on_tool_use do?**
- **Answer**: If True, agent can reconsider after tool calls
- **Keywords**: Reflection, tool result analysis

**Q34: How do you add tools to an agent?**
- **Answer**: Pass tools list to AssistantAgent(tools=[...])
- **Keywords**: Tool registration, parameter

**Q35: What happens if tool_call fails?**
- **Answer**: ToolCallSummaryMessage contains error; agent sees error
- **Keywords**: Error handling, recovery

### Team Configuration

**Q36: What does termination_condition do?**
- **Answer**: Stops team when condition met (e.g., stop word seen)
- **Keywords**: Termination logic, stopping condition

**Q37: What is StopMessageTermination?**
- **Answer**: Stops when message contains specific word
- **Keywords**: Content-based termination

**Q38: What is MaxMessageTermination?**
- **Answer**: Stops after N total messages
- **Keywords**: Count-based termination, safety

**Q39: How do you create custom termination?**
- **Answer**: Inherit TerminationCondition, implement __call__
- **Keywords**: Custom logic, extensibility

**Q40: Can you combine termination conditions?**
- **Answer**: Yes, OR logic (stop if any triggered)
- **Keywords**: Composition, multiple conditions

### Streaming

**Q41: What event types does run_stream() emit?**
- **Answer**: agent_starting, message_created, tool_call_starting, tool_call_completed, team_finished
- **Keywords**: Event types, async iteration

**Q42: How do you detect when streaming finishes?**
- **Answer**: Look for team_finished event
- **Keywords**: Completion detection, event type

**Q43: Can you interrupt a stream?**
- **Answer**: Yes, break from async for loop
- **Keywords**: Cancellation, interruption

**Q44: What's the difference between run() and run_stream()?**
- **Answer**: run() blocks until done, run_stream() yields events in real-time
- **Keywords**: Blocking vs streaming, latency

**Q45: When would you use run_stream()?**
- **Answer**: Need real-time updates, long-running tasks, monitoring
- **Keywords**: Real-time feedback, visibility

### Tool Definition

**Q46: How do you define a tool?**
- **Answer**: Create Tool(name, func, description)
- **Keywords**: Tool definition, schema

**Q47: What should a tool's docstring contain?**
- **Answer**: Description of what it does, parameter descriptions
- **Keywords**: Documentation, schema inference

**Q48: What types can tool parameters have?**
- **Answer**: str, int, float, bool, list, dict
- **Keywords**: Type hints, schema

**Q49: Can tools return complex types?**
- **Answer**: Best as string (LLM can parse); agent sees as text
- **Keywords**: Serialization, string representation

**Q50: How do you handle tool errors?**
- **Answer**: Raise exception; agent sees error in ToolCallSummaryMessage
- **Keywords**: Error propagation, exception handling

### State Management

**Q51: What is save_state()?**
- **Answer**: Serializes agent state to file
- **Keywords**: Persistence, serialization

**Q52: What is load_state()?**
- **Answer**: Deserializes state from file, resumes agent
- **Keywords**: Restoration, deserialization

**Q53: What can be persisted?**
- **Answer**: Messages, metadata, agent state
- **Keywords**: Stateful components

**Q54: What should NOT be persisted?**
- **Answer**: Credentials, API keys, temporary cache
- **Keywords**: Security, sensitivity

**Q55: Can you resume from state with different model_client?**
- **Answer**: Yes, but behavior may differ
- **Keywords**: Model compatibility, state transfer

### Observability

**Q56: How do you track tokens?**
- **Answer**: Most model_clients expose token counts in results
- **Keywords**: Token tracking, cost

**Q57: How do you measure latency?**
- **Answer**: time.time() before/after, or analyze events from stream
- **Keywords**: Performance measurement

**Q58: How do you log errors?**
- **Answer**: Catch exceptions, use logging module, structured logs
- **Keywords**: Logging, debugging

**Q59: What should you log?**
- **Answer**: request_id, error_type, agent, tool, attempt, recovery
- **Keywords**: Structured logging, context

**Q60: How do you trace a request through team?**
- **Answer**: Use correlation ID (request_id) in logs
- **Keywords**: Tracing, correlation

---

## Level 3: Debugging (15 Questions)

**Q61: Agent loops forever. How do you debug?**
- **Check**: max_turns, termination_condition, agent logic
- **Fix**: Add termination condition with stop words
- **Verify**: Test with short max_turns first

**Q62: Tool called 20 times. Is that bad?**
- **Check**: Is agent stuck in loop? Or legitimate tool use?
- **Investigate**: Look at tool results—are they changing?
- **Fix**: If stuck, add max_tool_iterations limit

**Q63: Model produces invalid output. What happened?**
- **Check**: Is output matching expected format?
- **Investigate**: Does agent see the error? Can it recover?
- **Fix**: Improve prompt, add validation, use function calling

**Q64: Handoff to non-existent agent. What do you do?**
- **Check**: Agent names in team match handoff targets
- **Fix**: Add default agent or validation
- **Verify**: Test handoff routes

**Q65: State can't deserialize. What's wrong?**
- **Check**: Is state format compatible?
- **Fix**: Clear state, start fresh, or implement migration
- **Prevent**: Version state properly

**Q66: Tool timeout after actually completing. How?**
- **Problem**: Request timed out but side effect happened
- **Solution**: Implement idempotency keys
- **Detect**: Check for duplicates

**Q67: Selector always chooses same agent. Why?**
- **Check**: Selector prompt and available agents
- **Fix**: Improve selector prompt to differentiate agents
- **Verify**: Manually check selection logic

**Q68: Team terminates early. Why?**
- **Check**: Termination condition triggered early?
- **Fix**: Adjust stop words or termination logic
- **Verify**: Test with verbose logging

**Q69: Tool returns wrong format. What to do?**
- **Check**: Tool implementation vs expected format
- **Fix**: Update tool to return correct format
- **Workaround**: Parse tool result more flexibly in agent

**Q70: API rate limit hit. How to recover?**
- **Immediate**: Retry with exponential backoff + jitter
- **Short-term**: Switch to cheaper model
- **Long-term**: Implement circuit breaker, queue requests

**Q71: Context overflow. What to do?**
- **Immediate**: Summarize old messages, keep recent
- **Short-term**: Switch to larger context model
- **Long-term**: Implement context management

**Q72: Human approval gets stuck. Why?**
- **Check**: UserProxyAgent blocking correctly?
- **Fix**: Provide input, or use timeout
- **Alternative**: Use HITL without blocking

**Q73: Tool says "Permission denied". Now what?**
- **Check**: API credentials correct?
- **Fix**: Update credentials, check permissions
- **Escalate**: May need manual intervention

**Q74: Repeated tool calls but results unchanged. Stuck?**
- **Yes**: Agent not making progress
- **Fix**: Add loop detection, force termination
- **Debug**: Why is agent not recognizing completion?

**Q75: Model refusal. How to continue?**
- **Immediate**: Try rephrased request once
- **If still refused**: Escalate to human, acknowledge limitation
- **Don't**: Keep retrying same request

---

## Level 4: System Design (15 Questions)

**Q76: Design a research system that survives external API failures.**

**Architecture**:
```
Researcher (primary) → fallback cache
Analyzer (no external dependency)
Writer (no external dependency)
```

**Failure handling**:
- Tool timeout → retry → fallback cache → error message
- Circuit breaker on web API

**Q77: Design a multi-agent customer support system.**

**Agents**:
- Frontline: resolve simple issues
- Specialist: complex issues
- Escalation: human hand-off

**Routing**: Swarm with handoffs

**HITL**: UserProxyAgent for approval

**Q78: Design an agent that learns from feedback.**

**Memory**: Store facts from conversations
**Learning**: Update memory based on user corrections
**Retrieval**: Use memory to inform next response

**Q79: Design approval workflow with human-in-loop.**

**Flow**:
- Agent produces → Human reviews (UserProxyAgent) → Decision

**State**: Save checkpoint before human review
**Timeout**: Use timeout for human input
**Fallback**: If human unavailable, escalate

**Q80: Design cost-controlled agent system.**

**Budget tracking**: Monitor tokens per request
**Fallback models**: Use cheaper model if cost high
**Queuing**: Queue requests if over rate limit
**Alerting**: Alert when 80% of budget spent

**Q81: Design multi-agent system with observability.**

**Logging**:
- Structured logs with request_id
- Error classification
- Tool calls and results

**Metrics**:
- Success rate
- Token usage
- Latency (p50, p95, p99)

**Dashboards**: Real-time monitoring

**Q82: Design agent evaluation framework.**

**Levels**:
1. Output quality
2. Agent behavior
3. Tool usage
4. Trajectory
5. Multi-agent coordination
6. End-to-end

**Metrics**: Success rate, quality score, latency, cost

**Q83: Design fallback strategy for mission-critical task.**

**Tier 1**: Primary model + tools
**Tier 2**: Fallback model
**Tier 3**: Cached data
**Tier 4**: Human escalation

**Q84: Design system that prevents infinite loops.**

**Prevention**:
- max_turns limit
- Termination condition with keywords
- Loop detection algorithm
- Forced termination after N repetitions

**Q85: Design agent system with state persistence.**

**Checkpoint**: Save after each major stage
**Versioning**: Version state for backward compatibility
**Recovery**: Resume from checkpoint on failure

**Q86: Design secure agent system.**

**Sandboxing**: Tools run in isolated environment
**Input validation**: Reject malicious inputs
**Output filtering**: Remove sensitive data
**Audit**: Log all actions with request_id

**Q87: Design agent for real-time decision making.**

**Speed**: Use streaming, small context
**Fallback**: Pre-computed responses for common cases
**Quality**: Accept slightly lower quality for speed

**Q88: Design multi-tenant agent system.**

**Isolation**: Each tenant has separate state
**Resources**: Per-tenant rate limits
**Cost**: Track cost per tenant
**Privacy**: Prevent data leakage between tenants

**Q89: Design agent that scales to 1000 concurrent users.**

**Concurrency**: Stateless agent design
**Queuing**: Queue requests, process in batches
**Caching**: Cache tool results
**Distributed**: Run on multiple machines

**Q90: Design agent with A/B testing framework.**

**Control**: Baseline version
**Test**: New version with different prompt/model
**Metrics**: Compare success rate, quality, cost
**Rollout**: Route X% traffic to test version

---

## Level 5: Production & Evaluation (10 Questions)

**Q91: How do you handle model provider outage?**
- **Immediate**: Fail over to secondary provider
- **Short-term**: Use cached responses
- **Long-term**: Implement circuit breaker, multi-provider setup

**Q92: What SLO would you set for agent system?**
- **Success rate**: 95%+
- **Latency p95**: < 10 seconds
- **Cost per request**: < $0.50
- **Availability**: 99.5%

**Q93: How do you evaluate multi-agent system?**
- **Success rate**: Task completion
- **Quality**: Final answer correctness
- **Efficiency**: Unnecessary steps
- **Coordination**: Proper handoffs
- **Cost**: Tokens used

**Q94: How would you detect that an agent has degraded?**
- **Monitor**: Success rate, quality metrics
- **Alert**: If success_rate drops > 2%
- **Investigate**: Golden dataset regression testing
- **Recovery**: Rollback to previous version

**Q95: What metrics matter most in production?**
1. **Success rate** (is it working?)
2. **Error rate** (is it failing?)
3. **Latency p95** (is it fast enough?)
4. **Cost** (is it affordable?)
5. **SLO compliance** (are we meeting targets?)

**Q96: How do you A/B test agent improvements?**
- **Control**: Current version (80% traffic)
- **Test**: New version (20% traffic)
- **Metrics**: Compare success rate, quality, latency, cost
- **Duration**: Run for 1 week minimum
- **Decision**: Rollout if metrics better or equal

**Q97: How do you implement graceful degradation?**
- **Full**: Research + Analysis + Review + Writing
- **Degraded 1**: Skip review (less quality)
- **Degraded 2**: Skip analysis (minimal processing)
- **Minimal**: Respond based on researcher only

**Q98: How would you handle a critical bug discovered in production?**
1. **Immediate**: Rollback to previous version
2. **Communicate**: Notify users of issue
3. **Debug**: Understand root cause
4. **Fix**: Implement fix
5. **Test**: Verify with golden dataset
6. **Redeploy**: Roll forward with fix

**Q99: What's the difference between quality metrics and operational metrics?**
- **Quality**: Does output meet goals? (correctness, completeness)
- **Operational**: Does system run? (uptime, latency, cost)
- **Both matter**: Good quality with poor ops = fail, vice versa

**Q100: When should you escalate to human instead of retry?**
- **After retries fail**: 3+ retries without recovery
- **Safety issue**: Policy violation detected
- **Business decision**: Needs human judgment
- **Cost**: Too expensive to continue automatically

---

## Rapid-Fire Interview Prep

### Concept → Definition

| Concept | One-Line Definition |
|---------|-------------------|
| Agent | Autonomous system that perceives, reasons, and acts |
| Team | Multiple agents coordinated for a goal |
| Tool | Function an agent can call |
| Handoff | Agent transferring control to another agent |
| Termination | Stopping condition for team execution |
| Tool iteration | Agent calling tool multiple times |
| Retry | Trying same request again after failure |
| Fallback | Using alternative approach after failure |
| Graceful degradation | System continues with reduced functionality |
| SLO | Target metric (99% success rate) |
| SLA | Contract promising SLO (refund if missed) |

### Common Wrong Answers

| Q | Wrong | Right |
|---|-------|-------|
| "What's retry?" | "Try again until success" | "Retry transient failures with backoff" |
| "When retry?" | "Always" | "Only transient failures (timeout, rate limit)" |
| "What's fallback?" | "Try retry" | "Try completely different approach" |
| "What's team?" | "Multiple models" | "Multiple agents (can use same model)" |
| "What's state?" | "Message history" | "Current snapshot (separate from history)" |
| "What's max_turns?" | "Max tool calls" | "Max agent turns (sequential steps)" |

---

## Cheat Sheets

### Production Checklist (Copy-Paste)

```
RELIABILITY
□ Models have fallback (primary → secondary → tertiary)
□ Tools have timeout
□ Tools have retry (if idempotent)
□ Circuit breaker for flaky APIs
□ Error classification (transient vs permanent)

FAILURE HANDLING
□ All error types identified
□ Retry strategy defined per error
□ Fallback strategy defined
□ Human escalation path clear
□ Graceful degradation tested

OBSERVABILITY
□ Structured logging with request_id
□ Error logs include context (agent, tool, attempt)
□ Metrics collection (success rate, latency, cost)
□ Dashboards for ops team
□ Alerting rules configured

TESTING
□ Golden dataset created
□ Failure scenarios tested
□ Retry logic tested
□ Fallback path tested
□ State recovery tested

DEPLOYMENT
□ Gradual rollout plan (5% → 25% → 100%)
□ Monitoring enabled before deploy
□ Alert thresholds tuned
□ Rollback procedure documented
□ On-call runbook prepared

OPERATIONS
□ Error handling documented
□ On-call runbook created
□ SLO/SLA defined
□ Cost budget set
□ Alert on-call rotation established
```

### Interview Talking Points

```
FUNDAMENTALS
"AutoGen is a multi-agent framework where LLMs make decisions,
use tools, and coordinate with each other."

TEAMS
"Different orchestration strategies: RoundRobin (fixed order),
Selector (AI-decided), Swarm (agent-decided handoff)."

FAILURES
"Classify failures (transient vs permanent), retry transient
failures with exponential backoff, use fallback for permanent."

EVALUATION
"Evaluate at multiple levels: output quality, agent decisions,
tool usage, trajectory, multi-agent coordination, end-to-end."

PRODUCTION
"Production requires: fallbacks, retry logic, timeouts,
circuit breakers, observability, metrics, graceful degradation."
```

---

## References

- [15-termination-cancellation.md](15-termination-cancellation.md) - Termination logic
- [20-human-in-loop.md](20-human-in-loop.md) - Human escalation
- [22-performance-cost.md](22-performance-cost.md) - Cost metrics
- [25-complete-project.md](25-complete-project.md) - Production examples
- [autogen-production-strategies.md](autogen-production-strategies.md) - Failure handling
