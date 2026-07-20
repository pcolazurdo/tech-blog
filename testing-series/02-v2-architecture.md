# The V2 Architecture: Observability, Guardrails, and Token Economics

*Part 2 of 3: How tracing revealed the real problems and what we changed*

---

After the initial prototype proved the concept worked — AI agents can analyze a system, generate tests, and execute them — I spent the next week watching it fail in interesting ways. The architecture worked *in theory*, but the operational reality was different: sessions crashed silently, sub-agents re-did each other's work, and the generated test scripts couldn't authenticate.

This article covers the V2 architecture that emerged from debugging those failures. The theme: **you can't fix what you can't see**.

## Adding Observability First

Before making any architectural changes, I added three observability layers:

### 1. OpenTelemetry Tracing

Every agent gets `trace_attributes` for correlation:
```python
config_analyzer = create_config_analyzer(
    ...,
    trace_attributes=agent_trace_attributes("config_analyzer", session_id),
)
```

The Strands SDK emits spans at four levels: agent → cycle → model invocation → tool invocation. With a Jaeger instance running locally:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 testing-agent run --trace "test leave"
```

### 2. Per-Agent Token Metrics

After every sub-agent completes, we log usage:
```
[2026-05-17 08:40:13] [Metrics] agent=config_analyzer tokens={input: 273612, output: 3215, total: 276827}
```

This immediately revealed the problem: a single Config Analyzer call was consuming 273K input tokens. The Scenario Generator was calling it 50+ times.

### 3. Session-Scoped Logs

Every session produces three files:
- `session-{id}.log` — full agent reasoning trace (streaming output)
- `http-requests-{id}.log` — every API call to the target system
- `debug-{id}.log` — structured debug logging (when `--debug` flag set)

## What Tracing Revealed

Looking at the Jaeger traces for a typical session, I saw:

```
orchestrator (turn 1)
  └── analyze_configuration
        └── config_analyzer
              ├── fetch_config → 200 (45ms)
              ├── query_endpoint → 200 (32ms)
              ├── query_endpoint → 200 (28ms)
              └── ... (15 more calls)
              
orchestrator (turn 2)
  └── generate_scenarios
        └── scenario_generator
              ├── query_config_analyzer → config_analyzer (8 tool calls, 500 lines)
              ├── query_config_analyzer → config_analyzer (12 tool calls, 600 lines)
              ├── query_config_analyzer → config_analyzer (6 tool calls, 400 lines)
              └── ... (47 more calls)
              └── TRUNCATED — max_tokens hit
```

The Scenario Generator's prompt said "GATHER CONTEXT: Use the Config Analyzer liberally." So it did — calling it 50 times, each call triggering a fresh analysis that produced hundreds of lines. The combined output exhausted the orchestrator's 4096 token budget, and the session stopped mid-sentence.

> **🔑 Lesson Learned: Token Exhaustion from Redundant Sub-Agent Calls**
>
> The token limit is hit at the model layer — Bedrock returns `stop_reason: "max_tokens"` and the response is simply truncated. No exception, no error message. The session log just stops. This is a *silent failure mode* that's impossible to diagnose without per-agent token metrics.
>
> The root cause: the Scenario Generator was re-gathering data that the orchestrator had already analyzed and passed forward. Sub-agent prompts must say "use what you were given" — not "go gather everything yourself."

## V2: The Architectural Responses

### Response 1: Context-First Scenario Generation

Instead of letting the Scenario Generator re-query the Config Analyzer, the orchestrator now passes the full analysis as context:

```
BEFORE: Scenario Generator → calls Config Analyzer 50x → produces plan
AFTER:  Orchestrator passes analysis → Scenario Generator works from it (max 3 follow-ups)
```

The Scenario Generator's prompt was rewritten:
```
IMPORTANT: The orchestrator has already run a full configuration analysis and passed
the results to you in the prompt. Do NOT call query_config_analyzer to re-gather data
that is already present. Limit to 3 calls maximum.
```

### Response 2: Config Analyzer Conciseness

The Config Analyzer was producing 500+ line narrative reports. For user display, this is fine. For inter-agent communication, it wastes tokens. Added output format instructions:

```
Output format:
- Keep responses CONCISE. Use structured tables and short bullet points.
- Target 50-100 lines maximum per response.
- Return raw facts and values — the orchestrator will synthesize for the user.
```

### Response 3: Orchestrator Token Budget

Bumped `max_tokens` from 4096 to 16384. The orchestrator holds the full conversation history and needs room for long-running sessions.

### Response 4: Header Stripping

HTTP responses from the target system included noise headers (`content-length`, `cache-control`, `set-cookie`, `connection`) that consumed context without providing value. Stripped at the client level:

```python
STRIPPED_RESPONSE_HEADERS = {"content-length", "cache-control", "set-cookie", "connection"}
```

## The Authentication Problem

Generated test scripts kept failing with 401 errors. The diagnosis took longer than it should have:

1. Scripts were hardcoding `BASE_URL` and `API_KEY` instead of reading from environment variables
2. A `.netrc` file on the developer machine was injecting conflicting auth headers
3. The executor was checking `if api_key:` which is falsy for empty strings

Three fixes:
```python
# 1. Auth snippet injected into test executor's system prompt
session.trust_env = False  # Ignore .netrc

# 2. NEVER hardcode — prompt says CRITICAL in all caps
"NEVER hardcode BASE_URL or API_KEY values in the script."

# 3. Check None, not truthiness
if context.auth_config.get("api_key") is not None:
    env["TEST_API_KEY"] = context.auth_config["api_key"]
```

## Script Persistence

Every generated script is saved to `.state/scripts/{scenario_id}.py` before execution. This solved two problems:
1. When a script crashes, you can inspect exactly what ran
2. Scripts can be re-executed manually for debugging

## The WorkItem Problem

The Scenario Generator used a "work item" model — structured objects dispatched to parallel workers. In practice, every validation error was a different failure mode:

```
- "multi_property" for assignment_type (enum only allowed 3 values)
- String for dependency_info (expected dict)
- Dict for additional_context (expected string)
```

The LLM kept generating values that didn't fit the schema. Each fix created a new surface for the next crash.

> **🔑 Lesson Learned: Don't Put Strict Schemas at LLM-to-LLM Boundaries**
>
> WorkItem sat between the Planner LLM and the Worker LLM. Imposing a rigid Pydantic schema on data that is inherently free-form LLM output creates fragility. We relaxed it to `ConfigDict(extra="allow")` with `Any`-typed fields, but this was papering over a deeper problem — the architecture was wrong.

## Natural Language Output Mode

Not every user wants to execute tests directly. Some want a test plan document they can hand to an RPA tool or a manual testing team. Added `--output nl` mode:

```bash
testing-agent run --output nl "test leave workflow"
```

This produces a structured markdown document saved to `.state/test-plan-{session_id}.md` — with descriptions, preconditions, steps, and expected outcomes for each test.

The implementation is clean: when `output_mode == "nl"`, the orchestrator gets an `export_test_plan` tool instead of `execute_tests`. Same workflow, different final step.

## V2 Architecture Summary

```
┌──────────┐     ┌──────────────────────────────────┐     ┌──────────────┐
│   User   │◄───►│   Orchestrator (one phase/turn)  │────►│ Target System│
└──────────┘     └────────────────┬─────────────────┘     └──────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
        ┌──────────┐       ┌──────────┐       ┌──────────┐
        │  Config  │       │ Scenario │       │   Test   │
        │ Analyzer │       │Generator │       │ Executor │
        │(READ-ONLY│       │(context- │       │(auth via │
        │ enforced)│       │  first)  │       │ env vars)│
        └──────────┘       └──────────┘       └──────────┘

Observability: OTLP traces + per-agent metrics + session logs + debug logs
```

Key V2 properties:
- **One phase per turn** — structurally enforced
- **Read-only analysis** — code-enforced, not just prompt
- **Context-first generation** — no redundant sub-agent calls
- **Token visibility** — per-agent metrics catch budget issues
- **Auth isolation** — credentials via env vars, `trust_env=False`
- **Script persistence** — every script saved before execution

## What Still Didn't Work

Even with all these fixes, the worker fan-out architecture was problematic:
- WorkItems carried too much context that polluted downstream agents
- The rigid schema kept breaking with creative LLM values
- Parallel workers added complexity without clear benefit for 10-15 test scenarios
- Each worker's output had to be aggregated — another failure point

In Part 3, I'll cover the complete rethink: removing workers entirely in favor of a sequential test plan architecture with an in-memory PlanStore.

---

*This is Part 2 of a 3-part series. [← Part 1: The Idea](01-building-an-ai-testing-agent.md) | [Part 3: Sequential Plan Architecture →](03-sequential-plan-architecture.md)*
