---
name: prompt-engineering-agent
description: "Design tool-using agent loops: Plan → Execute → Observe → Adapt. Covers system prompts for agents, tool-use prompt format, recovery prompts, planning prompts, the atoms→molecules→structures composition model, and step budgets. Use when building an agent that must take multiple actions, call tools, or iterate to reach a goal."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Pairs with provider tool-use APIs, MCP, and orchestration libraries (LangGraph, Mastra, custom DAGs)."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep Bash
---

# Agent Prompts

> **Disclaimer.** Tool-use APIs, parallel tool calling, and computer-use APIs vary by provider and evolve quickly. The prompt patterns here are stable; the wire format is the provider's documentation.

A chatbot answers a question. An agent **takes actions** to achieve a goal — calls tools, observes results, decides what to do next. The agent loop is `Plan → Execute → Observe → Adapt`, and the prompts that drive it are the difference between a useful agent and a model that flails for 30 turns and gives up.

## The composition model

```text
Prompts are atoms.
Skills are molecules.
Agents are structures.
```

An agent is built from **layers** of prompts:

- **System prompt** — identity, constitution, hard rules. Stable.
- **Planning prompt** — break the goal into steps. Per-task.
- **Tool-use prompts** — when and how to call which tool. Per-step.
- **Recovery prompts** — what to do when a tool fails or the plan stalls. Per-failure.
- **Reflection prompt** — has the goal been met? End or continue?

This is also the test surface. Each layer has its own fixtures (`prompt-engineering-eval`).

## The agent loop

```text
plan      = call(planning_prompt, goal, context)
while not done and steps < budget:
    action = call(action_prompt, plan, history)
    if action.is_tool_call:
        result = execute(action.tool, action.args)
        history.append((action, result))
    else:
        history.append(action)        # internal reasoning step
    if reflect(history).complete:
        return action.answer
    if reflect(history).stalled:
        plan = replan(planning_prompt, goal, history)
return partial_result(history)        # budget exhausted
```

Three things make this loop production-grade:

1. **A step budget** — never unbounded.
2. **Validation between steps** — schema-checked tool calls, schema-checked tool results.
3. **A recovery path** — every failure mode has a planned response.

## The agent system prompt

Same five-part structure as `prompt-engineering-system-prompt` (Identity / Capabilities / Limitations / Behavior / Format), with three additions:

```text
# Identity, Capabilities, Limitations, Behavior, Format  (as usual)

# Tools
You have access to these tools. Use them when their precondition is met.
Always prefer reading (search_kb, get_account) before writing (open_ticket, send_email).

<tool name="search_kb">
  <description>Search the knowledge base. Idempotent.</description>
  <input_schema>{"query": "string", "limit": "integer (1..10, default 5)"}</input_schema>
</tool>
... (≤ 7 tools is the sweet spot)

# Loop discipline
- Plan before acting. State your plan in <plan>...</plan> on the first turn.
- After each tool call, briefly note what you observed and whether the plan still holds.
- If the same tool call fails twice with similar errors, STOP retrying and replan.
- If you've used 80% of your step budget without progress, summarize what you've learned and ask the user.

# Stop conditions
- The goal is met → return <final>...</final>.
- The user's request is unsafe or out of scope → return <refuse reason="..."/>.
- The plan is unrecoverable → return <abort summary="..." next_step="..."/>.
```

## Tool-use prompt format

Two reliable shapes. Pick one for the whole agent.

**JSON tool calls** (provider-native on most platforms):

```json
{ "tool": "search_kb", "args": { "query": "duplicate charge June", "limit": 5 } }
```

**XML tool calls** (Claude-friendly, easy to embed in prose):

```xml
<tool name="search_kb">
  <args>{"query": "duplicate charge June", "limit": 5}</args>
</tool>
```

When the provider has native tool-use, **use it.** Native tool-use enforces the schema, removes parsing errors, and supports parallel tool calls. The prompt patterns above describe the **agent's reasoning around** tool calls; the wire format follows the provider.

## Tool design rules

- **Few, well-named tools.** 5–7 is ideal; >12 confuses the model.
- **Idempotent reads, confirmed writes.** A `delete` tool should require a confirmation step or a separate prompt.
- **Schema-first.** Every tool has a JSON Schema input/output. Validate both.
- **One responsibility per tool.** A `do_billing_thing(action, …)` tool is a footgun; split.
- **Errors are part of the API.** Return structured errors (`{"error": "...", "retryable": false}`) so the agent can adapt.

## Planning prompts

Make planning explicit. Before any tool call:

```text
You will receive a goal. Output ONLY a plan in this shape:

<plan>
  <step n="1" tool="..." reason="..."/>
  <step n="2" tool="..." reason="..."/>
  ...
</plan>

Constraints:
- ≤ 5 steps in the initial plan.
- Each step names exactly one tool OR is a reasoning step.
- Plans are revisable; you'll be asked to update after each result.
```

A visible plan gives you something to evaluate. A plan in the model's head is opaque — when it goes wrong, you can't tell whether the plan was wrong or the execution was.

## Recovery prompts

When a tool fails, the agent needs a structured way to recover. Don't let it just retry blindly.

```text
A tool call returned an error:

<failed_call>
  <tool>...</tool>
  <args>...</args>
  <error>...</error>
  <retryable>true|false</retryable>
</failed_call>

Decide ONE of:
1. RETRY — same call (only if retryable=true and this is the first retry).
2. ADJUST — different args, explain what changed.
3. ALTERNATIVE — different tool that achieves the same sub-goal.
4. REPLAN — the plan no longer works; produce a new <plan>.
5. ABORT — return <abort summary="..." next_step="..."/>.

Output: <decision kind="...">...</decision>
```

Forcing the choice into a closed set prevents the model from inventing creative-but-wrong recovery (e.g. fabricating fake credentials when an auth tool fails).

## Reflection / stop prompts

After each step:

```text
Given the history below, output exactly one of:
- <continue>...next sub-goal...</continue>
- <complete answer="...">...</complete>
- <stalled reason="..."/>

A goal is complete when:
- All requested information has been gathered AND
- All requested actions have succeeded AND
- The user-visible answer addresses the original request.

A goal is stalled when:
- The same sub-goal has been attempted ≥3 times without progress, OR
- The remaining step budget is < the optimistic step count for the remaining work.
```

## Step budgets

Every agent invocation has:

- A **step budget** (max tool calls + reasoning steps). Default 10–25.
- A **time budget** (wall-clock).
- A **token budget** (cumulative input + output).

Hit any one → graceful exit:

```text
<abort summary="What I learned and didn't finish" next_step="What the user can try"/>
```

Without budgets, agents loop. Loops are how prototypes become outages.

## State and memory

The agent's "memory" is the conversation history fed back each turn. As history grows, attention degrades. Mitigate with:

- **Summarize older turns** beyond a window (`prompt-engineering-context`).
- **Persist key facts** to a tool (`remember(key, value)`) the agent reads back when needed.
- **Trim tool results** that are no longer needed for the current sub-goal.

For long-running agents, treat history as RAM — evict the irrelevant.

## Multi-agent / sub-agents

Spawn a sub-agent when:

- A sub-task has a different role / system prompt than the parent.
- Parallel work is independent and the merge is well-defined.
- Context from the sub-task is too large to fold into the parent's history.

Don't spawn a sub-agent when:

- The parent could do it in one prompt.
- Coordination overhead exceeds the speedup.
- You can't enumerate the sub-agent's stop conditions.

Sub-agents have their own step budgets and their own evals.

## Observability

Log per turn:

- Step number, plan, action (tool + args), result, latency, tokens.
- Any retry decisions and their reasons.
- Final stop condition (`complete` / `stalled` / `aborted` / `budget_exhausted`).

Dashboards: % of runs hitting each stop condition; average steps per goal; tool error rate per tool; budget consumption distribution.

## Anti-patterns

- **No step budget.** Loop indefinitely → cost runaway, user waiting.
- **Free-text tool calls.** Parser failures, hallucinated tools, wrong arg names. Use provider tool-use or strict schemas.
- **Too many tools.** A tool palette of 30 spreads attention thin.
- **No plan before action.** Black-box agents are uneditable when they go wrong.
- **Same retry on failure.** Recovery prompts force a structured choice.
- **Persistent history with no eviction.** Quality and cost both degrade with turn count.
- **Silent failures.** Always return one of `complete | stalled | aborted | budget_exhausted` so the caller knows.
- **No eval per layer.** Test the planning prompt, the recovery prompt, and the end-to-end loop separately.

## Quick template

```text
SYSTEM:
  identity + capabilities + limitations + behavior + format
  + tool list (schema-first, ≤7 tools)
  + loop discipline
  + stop conditions

USER (turn 1):
  goal: ...
  context: ...

AGENT (each turn):
  <plan>...</plan>          # turn 1 only, then revised on stall
  <thought>...</thought>    # short
  <tool name="..."><args>...</args></tool>
                            # OR
  <complete answer="..."/>

LOOP:
  step_budget = 15
  on tool error → recovery prompt → RETRY|ADJUST|ALTERNATIVE|REPLAN|ABORT
  on stall (≥3 sub-goal retries) → REPLAN once → otherwise ABORT
  on budget exhausted → ABORT with summary
```

## Cross-references

- `jimmy-skills@prompt-engineering-system-prompt` — base for the agent constitution.
- `jimmy-skills@prompt-engineering-context` — tools, MCP, history summarization.
- `jimmy-skills@prompt-engineering-chain` — chains are agents without tools.
- `jimmy-skills@prompt-engineering-output-json` — tool-call schemas.
- `jimmy-skills@prompt-engineering-edge-cases` — tool failure / adversarial input.
- `jimmy-skills@prompt-engineering-eval` — per-layer fixtures.
- `jimmy-skills@prompt-engineering-cost` — step budgets and tiering.
