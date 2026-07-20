# Building an AI-Powered Testing Agent for Enterprise SaaS

*Part 1 of 3: The idea, the goal, and the initial architecture*

---

Enterprise SaaS systems like Workday, SAP, and ERPNext aren't coded — they're *configured*. A leave policy, an approval chain, a payroll deduction rule — these are settings, not source code. And yet, validating that these configurations work correctly after a change still looks like it did 15 years ago: a subject matter expert opens a spreadsheet, writes test cases by hand, and clicks through the system for days.

I wanted to see if an AI agent could do this job. Not a chatbot that answers questions about testing — an *agent* that actually connects to a system, reads its configuration, generates a test plan, and executes it.

This is the story of building that system.

## The Problem Statement

When you change a configuration in an enterprise system, you need to validate three things:

1. **The change works** (happy path)
2. **Nothing else broke** (regression)
3. **Edge cases are handled** (boundary/negative)

Today, this validation depends on people who carry the knowledge in their heads. When they're busy, testing is skipped. When they leave, the knowledge goes with them.

The cost of a missed configuration error isn't abstract — it's incorrect salary payments, broken approval chains, or compliance violations. And by the time someone notices, it's already in production.

## The Hypothesis

If the system exposes APIs, an AI agent should be able to:
- Read the current configuration state
- Understand the relationships between settings
- Generate test scenarios that cover critical paths
- Execute those tests and report results

No access to source code needed. No test frameworks to install on the target. Just standard REST API access.

## Initial Architecture: Multi-Agent Orchestration

I chose a multi-agent architecture rather than a single monolithic agent for two reasons:

1. **Separation of concerns** — each agent has a focused role with scoped tools
2. **Token budget management** — each agent gets its own context window, preventing one phase from consuming the budget of another

Here's what I started with:

```
┌──────────┐     ┌──────────────────┐     ┌──────────────┐
│   User   │────►│   Orchestrator   │────►│ Target System│
└──────────┘     └────────┬─────────┘     └──────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Config  │ │ Scenario │ │   Test   │
        │ Analyzer │ │Generator │ │ Executor │
        └──────────┘ └──────────┘ └──────────┘
```

**Five agents, each with a distinct role:**

| Agent | Role | Tools |
|-------|------|-------|
| Orchestrator | Coordinates workflow, enforces approval gates | Dispatches to other agents |
| Config Analyzer | Reads and maps system configuration | `fetch_config`, `query_endpoint`, `parse_api_spec` |
| Scenario Generator | Plans tests using heuristics | Worker fan-out for parallel generation |
| Test Executor | Generates Python scripts and runs them | `generate_script`, `run_script` |
| Failure Analyst | Diagnoses test failures | `get_config_state`, `compare_responses` |

### Technology Choices

- **AWS Bedrock** for model access (Claude Sonnet 4.6 for most agents, Opus for failure analysis)
- **Strands SDK** as the agent framework — it handles tool registration, streaming, and conversation management
- **Python** throughout — the agents, the CLI, the infrastructure (CDK)

### The Workflow

The intended workflow was straightforward:

1. User makes a request: "Test the leave approval configuration"
2. Config Analyzer reads the system's current state
3. Scenario Generator creates test cases
4. User approves the plan
5. Test Executor generates scripts and runs them
6. Results are reported

Simple, right? In practice, every single one of these steps had surprises.

## The First Run

The first time I ran the system end-to-end, this happened:

1. The orchestrator called the Config Analyzer ✓
2. The Config Analyzer started making POST requests to the target system ✗
3. The orchestrator moved directly to execution without asking the user ✗
4. The session log was 1,200 lines and cut off mid-sentence ✗

Three fundamental architectural assumptions were wrong.

> **🔑 Lesson Learned: Prompt-Only Guardrails Are Not Enough**
>
> The system prompt said "always confirm with the user before executing tests" and "never modify the target system during analysis." The agent ignored both instructions because the *runtime architecture* gave it no mechanism to yield control back to the user (single-pass invocation) and no enforcement at the tool level (the HTTP client allowed any method).
>
> Guardrails must be structural (code-level), not just instructional (prompt-level). If the architecture allows an action, the model may take it regardless of what the prompt says.

## The Three Fixes That Defined the Architecture

### Fix 1: Multi-Turn Conversation Loop

The CLI was invoking the orchestrator as a single function call:
```python
response = orchestrator(session.user_request)
```

The Strands SDK's `Agent.__call__()` runs to completion in one turn. The model literally cannot stop mid-execution to wait for user input. So it plows through all phases regardless of prompt instructions.

The fix: a `while True` loop where each orchestrator call is one turn:
```python
prompt = session.user_request
while True:
    response = orchestrator(prompt)
    user_input = input("[You] ")
    prompt = user_input
```

This makes approval gates *structurally enforceable* — the model ends its turn, control returns to the user, and the user decides what happens next.

### Fix 2: Read-Only Config Analyzer

The Config Analyzer had a `query_endpoint` tool that accepted any HTTP method. The LLM decided to "probe" the system's behavior by making POST requests — creating actual records in the target system. From its perspective, this was clever analysis. From the user's perspective, it was unauthorized modification without approval.

The fix: enforce read-only at the *code level*, not just the prompt:
```python
def create_query_endpoint_tool(http_client, read_only=True):
    @tool
    def query_endpoint(path, method="GET", ...):
        if read_only and method.upper() not in ("GET", "HEAD"):
            return {"error": "Method not allowed in read-only mode..."}
```

### Fix 3: One Phase Per Turn

Even with the conversation loop, the orchestrator was chaining ANALYZING + GENERATING in a single turn. The Scenario Generator would then re-invoke the Config Analyzer 50+ times, each producing a 500-line report. This exhausted the token budget and the session crashed mid-output with no error message.

The fix: the system prompt explicitly says "You must ONLY call tools for ONE phase per turn." Combined with the multi-turn loop, this means:
- Turn 1: Analyze → present → stop
- Turn 2: Generate → present → stop
- Turn 3: Execute one test → present → stop

## What I Learned Building V1

1. **LLMs are creative problem-solvers — including when you don't want them to be.** The Config Analyzer wasn't malicious; it was being resourceful. "Read-only analysis" must be enforced at the tool level, not assumed from the role description.

2. **Token exhaustion is a silent failure mode.** Unlike exceptions, hitting `max_tokens` produces no error — the output just stops. This makes it hard to diagnose without observability.

3. **Sub-agents inherit the blast radius of their tools.** The orchestrator correctly avoided calling `execute_tests`, but a sub-agent with write access achieved the same effect through a different path. Every agent in the chain needs tool-level access control appropriate to its role.

4. **Multi-turn is not optional for approval gates.** If a workflow requires human checkpoints, the invocation pattern must support yielding control.

## What's Next

In Part 2, I'll cover the V2 architecture — how observability revealed the token budget problem, why the worker fan-out model didn't work in practice, and the shift to a sequential test plan architecture that actually executes reliably.

---

*This is Part 1 of a 3-part series on building an AI-powered testing agent. [Part 2: The V2 Architecture →](02-v2-architecture.md)*
