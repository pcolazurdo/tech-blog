# The Sequential Plan Architecture: Less is More

*Part 3 of 3: Replacing worker fan-out with a simpler model that actually works*

---

After two iterations of the Scenario Generator — first with a complex worker fan-out model, then with relaxed schemas to tolerate LLM creativity — I stepped back and asked: *what is this architecture actually buying us?*

The answer was: complexity. The worker model was designed for a future where we'd generate 50+ scenarios in parallel. In practice, we generated 10-15 tests, the workers kept crashing on schema validation, and the aggregation step was another place things could go wrong.

This article covers the third iteration: removing the entire worker machinery in favor of a simple list of test descriptions and a sequential execution loop.

## The Core Insight

The system has two phases with fundamentally different requirements:

1. **Planning** — decide *what* to test (creative, strategic, needs full context)
2. **Executing** — run each test (mechanical, needs only one test's description at a time)

The worker model conflated these: it tried to plan AND generate execution-ready structured objects in one pass. Every field on the WorkItem was a contract between the planner and the executor — and LLMs are terrible at honoring contracts.

The fix: separate them completely. The planner outputs a *natural language list*. The executor receives one item at a time and figures out how to test it.

## The New Architecture

```
Config Analyzer → analysis
        ↓
Test Planner → ["test_01: ...", "test_02: ...", ...]  (NL descriptions)
        ↓ (stored in PlanStore)
Orchestrator loop:
  for each test in plan:
      ↓
  Test Executor → generates script → runs it → reports result
      ↓
  mark_complete(test_id, "pass" | "fail" | "error")
```

### What Was Removed

- `src/agents/scenario_generator.py` — the planner/coordinator with fan-out
- `src/agents/scenario_worker.py` — parallel workers
- `WorkItem` model — the rigid typed contract between planner and workers
- `create_work_items`, `spawn_workers`, `aggregate_results` tools
- All the validators and coercions added to handle LLM-generated WorkItems

### What Was Added

- `src/agents/test_planner.py` — a simple agent with NO tools, just a system prompt
- `src/plan_store.py` — singleton that holds the test plan in memory
- Three new orchestrator tools: `get_next_test`, `mark_test_complete`, `get_test_plan`

## The Test Planner

The Test Planner is the simplest agent in the system. It has no tools — it just receives the configuration analysis and outputs a JSON array of test descriptions:

```python
SYSTEM_PROMPT = """\
You are the Test Planner. Given a configuration analysis, produce a structured
test plan — a numbered list of test descriptions that can each be executed
independently.

Each test must be SELF-CONTAINED — include all endpoint paths, parameter values,
and expected outcomes. The executor has no access to the original analysis.

Output ONLY the JSON array. No prose before or after.
"""

agent = Agent(model=model, system_prompt=SYSTEM_PROMPT, tools=[])
```

That's it. No `query_config_analyzer`. No `create_work_items`. No `spawn_workers`. The planner receives everything it needs in the prompt and returns a list.

### Sample Output

```json
[
  {
    "id": "test_01",
    "name": "create_employee_happy_path",
    "description": "Create an employee via POST to /api/resource/Employee with fields: first_name='Alice', last_name='Smith', company='pp', department='HR - P', designation='Manager', status='Active'. Expect HTTP 200 with a response containing the created employee name.",
    "category": "happy_path"
  },
  {
    "id": "test_02",
    "name": "circular_reports_to_rejected",
    "description": "Create two employees E1 and E2. Set E1.reports_to = E2. Then attempt to set E2.reports_to = E1. Expect HTTP 417 with a validation error indicating circular reference.",
    "category": "negative"
  }
]
```

Each description is self-contained: the Test Executor can generate a working script from it without any additional context.

## The PlanStore

PlanStore is a singleton that tracks the test plan and its execution progress:

```python
class PlanStore:
    _instance = None

    def set_plan(self, tests: list[dict], session_id: str | None = None) -> None: ...
    def get_next_test(self) -> dict | None: ...
    def mark_complete(self, test_id: str, status: str, result: str | None = None) -> None: ...
    def get_progress(self) -> dict: ...
    def get_plan_summary(self) -> str: ...
```

The summary output looks like:
```
Test Plan (5 tests):
  ✅ [test_01] create_employee_happy_path — pass
  ✅ [test_02] circular_reports_to_rejected — pass
  ❌ [test_03] cross_company_link_blocked — fail
  ⏳ [test_04] self_reporting_rejected — pending
  ⏳ [test_05] inactive_manager_rejected — pending

Progress: 3/5 (pass: 2, fail: 1, error: 0)
```

### Observability

PlanStore is instrumented with OpenTelemetry:

```python
from opentelemetry import trace
tracer = trace.get_tracer("testing-agent.plan_store")

def mark_complete(self, test_id, status, result=None):
    with tracer.start_as_current_span("PlanStore.mark_complete") as span:
        span.set_attribute("test.id", test_id)
        span.set_attribute("test.status", status)
        ...
```

In Jaeger, you see PlanStore spans nested alongside agent tool calls — full visibility into what happened when.

## The Execution Loop

The orchestrator now works in a tight loop: one test per turn, with the user in control.

**Orchestrator system prompt:**
```
3. EXECUTING: One test per turn:
   a. Call get_next_test to get the next pending test
   b. Call execute_tests with the test ID and description
   c. Call mark_test_complete with the result
   d. Present the result. STOP and ask "continue?" (GATE)
```

**What a turn looks like:**
```
[Orchestrator] Next test: test_03 — cross_company_link_blocked
  Calling execute_tests...

[Test Executor] Generating script for test_03...
  Running script...
  Result: FAIL — expected HTTP 417, got 200

[Orchestrator] Test test_03 FAILED.
  Expected: cross-company employee link should be rejected (HTTP 417)
  Actual: the system accepted it (HTTP 200)
  
  Progress: 3/5 (pass: 2, fail: 1)
  
  Would you like to continue to the next test?

[You] continue
```

## Why This Works Better

### 1. No Schema Contracts Between LLMs

The old model: Planner → WorkItem (typed schema) → Worker → TestScenario (typed schema)

The new model: Planner → natural language string → Executor

Natural language is what LLMs are good at. The Test Executor receives a description and generates whatever script it needs. No Pydantic models being misused as LLM-to-LLM contracts.

### 2. Each Turn is Self-Contained

The orchestrator sends one test description per turn to the executor. If it fails, you see the failure immediately. If the script has a bug, you can inspect it at `.state/scripts/{test_id}.py`. No batching, no aggregation, no cascading failures.

### 3. The User Sees Progress

Instead of waiting 5 minutes for all workers to complete and then seeing a wall of results, the user watches tests execute one at a time with clear pass/fail feedback. They can stop early if they've seen enough.

### 4. Token Budget is Predictable

Each turn has a bounded token cost: one orchestrator call + one executor call. No fan-out multiplication. No 50x Config Analyzer queries. The total cost for a 15-test session is predictable.

## The Skills System

To allow domain-specific customization without modifying agent code, each agent can load skills from a directory:

```
skills/
  general/                          ← shared by all agents
    api-testing-patterns/SKILL.md
  test_planner/                     ← planner-specific
  test_executor/                    ← executor-specific
```

Skills are loaded via the Strands `AgentSkills` plugin — only metadata goes into the system prompt; full instructions are lazy-loaded when the agent activates a skill. This keeps the base system generic while allowing domain specialization through files.

## Results

After the refactor, a typical session:
- **Config Analysis:** 1 turn, 30-60 seconds, produces a structured analysis
- **Test Planning:** 1 turn, 10-15 seconds, produces 10-15 test descriptions
- **Execution:** 10-15 turns, 30-60 seconds each, one test at a time
- **Total:** 8-15 minutes for complete coverage of a configuration area

No crashes. No token exhaustion. No schema validation errors. The user is in control at every step.

## What I'd Do Differently

If I were starting over:

1. **Skip the worker model entirely.** The sequential approach is simpler, more debuggable, and fast enough for 10-15 tests. Parallelism is premature optimization when your agents aren't reliable yet.

2. **Start with observability.** The OTEL tracing + per-agent metrics should have been day-one decisions. Every bug in V1 would have been found in minutes with proper traces instead of hours of session log archaeology.

3. **Use plain strings at LLM boundaries.** Typed schemas (Pydantic models, enums) are great for application code. They're terrible as contracts between LLMs. Natural language in, natural language out, with structure only at the edges.

4. **Make approval gates structural, not instructional.** Don't tell the model to stop — make the runtime yield control. The one-phase-per-turn design with the CLI loop is the most important architectural decision in the system.

## What's Next

The system works well locally. The next evolution is deploying it as a remote service on AWS AgentCore — where the CLI sends requests to a cloud-hosted agent runtime, and the multi-turn conversation happens over HTTP. The PlanStore singleton survives naturally in the AgentCore microVM, and skills are bundled in the container image.

But that's a story for another day.

---

*This is Part 3 of a 3-part series. [← Part 1: The Idea](01-building-an-ai-testing-agent.md) | [← Part 2: V2 Architecture](02-v2-architecture.md)*

---

## Series Summary

| Article | Key Theme |
|---------|-----------|
| Part 1 | The idea + why prompt-only guardrails fail |
| Part 2 | Observability reveals token economics + auth problems |
| Part 3 | Sequential plan replaces worker complexity |

**The meta-lesson across all three:** In agent systems, simplicity beats sophistication. The best architecture is the one where each failure is visible, each turn is bounded, and the user is never more than one interaction away from understanding what's happening.
