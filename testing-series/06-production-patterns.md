# Patterns for Production Agent Systems

*Part 6 of 6: Cross-cutting lessons from deploying multi-agent AI in production*

---

After six months of building, breaking, and rebuilding this platform — across three execution modes, two deployment targets, and dozens of debugging sessions — patterns emerged. Not theoretical best practices, but hard-won operational knowledge from watching agents fail in production.

This article distills those patterns. Some are obvious in hindsight. All cost time to learn the first time.

## Pattern 1: Defense in Depth for Agent Behavior

A single guardrail is never enough. We learned this three times:

**Prompt-only:** "Never modify the target system during analysis." → The agent made POST requests to "probe" behavior. Ignored.

**Prompt + tool enforcement:** Added `read_only=True` to `query_endpoint` which hard-blocks non-GET methods at the code level. Now the agent *can't* make writes even if it tries.

**Prompt + tool + architecture:** The one-phase-per-turn design means the orchestrator physically can't chain analysis → execution in a single invocation. The runtime yields control back to the user.

```
Layer 1: Prompt instructions (aspirational — LLMs may ignore)
Layer 2: Tool-level enforcement (code rejects invalid actions)
Layer 3: Architectural constraints (runtime structure prevents the action)
```

Apply this to every safety-critical behavior. If you only have Layer 1, you don't have a guardrail — you have a suggestion.

## Pattern 2: One Well-Informed Agent > Two Partially-Informed Agents

The browser testing feature proved this decisively. The original design:

```
ui_test_executor → browse_website tool → internal Agent → browser
```

The internal agent knew how to drive a browser but not *why* — it lacked auth credentials, test assertions, and the system prompt context. We paid for two LLM calls and got worse results than one.

The fix:
```
ui_test_executor → browser tool (directly)
```

One agent, full context, one reasoning loop. Every time you're tempted to create an "inner agent" inside a tool, ask: can the outer agent just use the tool directly?

**Exception:** When the inner agent needs a fundamentally different model (e.g., a cheaper model for bulk classification). But for same-model chains, eliminate the intermediary.

## Pattern 3: Observability Before Architecture

Every architectural problem in this project was an observability problem first:

| What We Saw | What Tracing Revealed | Time to Diagnose |
|-------------|----------------------|-----------------|
| "Session crashed mid-output" | Config Analyzer consumed 273K input tokens in one call | 30 seconds |
| "Tests fail with no error" | `_parse_script_output` was dropping `http_traces` from JSON | 2 minutes |
| "Auth doesn't work" | `.netrc` file injecting conflicting Basic auth header | 5 minutes |
| "Agent doesn't stop at gates" | Single-pass invocation — no mechanism to yield control | 10 minutes |

The tooling that made diagnosis fast:
- **Per-agent token metrics:** `[Metrics] agent=config_analyzer tokens={input: 273612, output: 3215}`
- **Session-scoped debug logs:** `testing-agent run --debug` → `.state/debug-{session_id}.log`
- **OTEL spans on PlanStore:** See exactly when tests are created, started, completed
- **HTTP request log:** Every API call to the target system, timestamped

Add observability on day one. Not "when we have time." Day one.

## Pattern 4: The PlanStore Pattern (In-Memory State with Tracing)

The PlanStore is a singleton that holds the test plan in memory:

```python
class PlanStore:
    _instance = None

    def set_plan(self, tests, session_id): ...
    def get_next_test(self) -> dict | None: ...
    def mark_complete(self, test_id, status, result): ...
    def get_progress(self) -> dict: ...
```

Why it works:
- **Singleton** — all tools in the orchestrator share the same state without passing references
- **In-memory** — no serialization overhead, no database round-trips
- **OTEL-instrumented** — every `mark_complete` emits a span with `test.id` and `test.status`
- **Survives across turns** — in both CLI (same process) and AgentCore (same microVM)

When to use this pattern: any state that multiple tools need to read/write within a session, where durability isn't critical (microVM death = start over).

## Pattern 5: Skills for Domain Specialization

The skills system (`skills/{agent_name}/SKILL.md` files) provides domain knowledge without modifying agent code:

```
skills/
  general/
    api-testing-patterns/SKILL.md   ← shared by all agents
  test_planner/
    leave-management/SKILL.md       ← planner-specific
  ui_test_executor/
    form-interaction/SKILL.md       ← UI executor-specific
```

Skills use the Strands `AgentSkills` plugin with progressive disclosure — only metadata goes into the system prompt; full instructions load on demand. This keeps base prompts small while allowing deep domain customization.

**Key insight:** Skills are the right mechanism for knowledge that changes per-client or per-domain (e.g., "this system uses Frappe's form patterns"). The system prompt is for behavior that's always true (e.g., "never hardcode credentials").

## Pattern 6: Security Hardening at Every Layer

| Concern | Implementation |
|---------|---------------|
| Bucket exposure | `block_public_access=s3.BlockPublicAccess.BLOCK_ALL` on every bucket |
| Credential leakage | `config/agents.yaml` gitignored; `.sample` file tracked |
| .netrc interference | `session.trust_env = False` in every generated script |
| Auth in scripts | Environment variables only (`TEST_API_KEY`), never hardcoded |
| Trace attributes | Sanitize: `if k.upper() not in ("UI_USERNAME", "UI_PASSWORD")` |
| Config delivery | S3-based (not in invocation payload — avoids CloudTrail logging secrets) |

The `.netrc` bug took hours to diagnose. A file on the developer's machine was injecting auth headers that conflicted with the script's own auth. `trust_env=False` is now in every auth snippet the system generates. One line of defense, permanently.

## Pattern 7: Centralized Model Configuration

Every agent needed a BedrockModel instance. The original code repeated this in 8 files:

```python
model = BedrockModel(
    model_id=config.model_id,
    region_name=region,
    temperature=config.temperature,
    max_tokens=config.max_tokens,
)
```

Two problems surfaced:
- Some models reject `temperature` or `top_p` when set (e.g., certain Bedrock model configurations return validation errors)
- Every new agent file copied the pattern, risking drift

The fix: a `bedrock_model_kwargs()` helper on the config model that omits None-valued params:

```python
class AgentModelConfig(BaseModel):
    model_id: str
    temperature: float | None = None  # None = let Bedrock decide
    max_tokens: int = 4096
    top_p: float | None = None

    def bedrock_model_kwargs(self, region_name: str = "us-east-1") -> dict:
        kwargs = {"model_id": self.model_id, "region_name": region_name, "max_tokens": self.max_tokens}
        if self.temperature is not None:
            kwargs["temperature"] = self.temperature
        if self.top_p is not None:
            kwargs["top_p"] = self.top_p
        return kwargs
```

Now every agent: `model = BedrockModel(**model_config.bedrock_model_kwargs(region))`

One line, no risk of sending unsupported params. When you don't have a reason to override a default, don't send the parameter at all.

## Pattern 8: The Sample File Pattern

Any config file that accumulates secrets or environment-specific values should be:
1. **Gitignored** — `config/agents.yaml` never in version control
2. **Tracked as `.sample`** — `config/agents.yaml.sample` with placeholder values
3. **Documented inline** — comments explain each field and valid values

```yaml
# config/agents.yaml.sample
target_system:
  base_url: http://localhost:8000
  auth_type: token  # "token" (Basic auth) or "api_key" (custom header)
  api_key_header: X-API-Key  # only used when auth_type: api_key

knowledge_base:
  knowledge_base_id: null  # set after running: make create-kb
```

New developers: `cp config/agents.yaml.sample config/agents.yaml` → fill in values → ready.

## Pattern 9: Makefile as Executable Documentation

```makefile
setup-kb: deploy-infra create-index create-kb update-config
deploy-agentcore: ./scripts/deploy_agentcore.sh
build-image: $(CONTAINER_RUNTIME) build --platform linux/arm64 ...
test: uv run python -m pytest tests/ -q
run: uv run testing-agent run
```

`make help` shows every operation the project supports. New team members don't read docs — they run `make help` and see what's possible. Each target is one command with a one-line description.

## Pattern 10: CDK for Infrastructure, Scripts for API Resources

We fought CDK for days trying to create an OpenSearch Serverless vector index as a custom resource. Lambda couldn't authenticate, timing issues with policy propagation, `cfnresponse` not available in Python 3.12 inline Lambdas.

The fix: CDK deploys *infrastructure* (buckets, roles, collections). Scripts create *API resources* (indexes, Knowledge Bases, data sources).

```makefile
setup-kb: deploy-infra create-index create-kb update-config
#         ↑ CDK        ↑ script    ↑ script  ↑ script
```

CDK is great for declarative infrastructure. It's terrible for imperative API calls that have timing dependencies and complex error handling. Use each tool where it's strong.

## Pattern 11: The Callback Handler Chain

When Agent A invokes Agent B which invokes Tool C which creates Agent D — where does output go?

```
Orchestrator (callback: _TeeCallbackHandler → session log)
  └── ui_test_executor (callback: same handler)
        └── browser tool (raw Strands tool — output streams through parent)
```

The rule: every agent in the chain must receive the same `callback_handler`. If you create an agent inside a tool, it must explicitly receive the handler — Strands doesn't inherit it.

We solved this with `set_browser_callback_handler()` initially, then eliminated the problem entirely by removing the internal agent. But for systems where nested agents are necessary, propagate the handler explicitly.

> **🔑 Lesson Learned: Every Operational Problem is an Observability Problem First**
>
> We never had a bug that was genuinely hard to fix. We had bugs that were hard to *see*. The moment we added per-agent token metrics, session-scoped debug logs, and OTEL tracing on PlanStore, diagnosis time dropped from hours to minutes. The fix was always obvious once the data was visible.
>
> If you're building a multi-agent system: instrument first, architect second. You can't fix what you can't see, and agents fail in ways that don't produce stack traces.

## The Meta-Pattern

Across all six articles, one theme:

**In agent systems, simplicity beats sophistication. The best architecture is the one where each failure is visible, each turn is bounded, and the user is never more than one interaction away from understanding what's happening.**

Worker fan-out was sophisticated. Sequential one-at-a-time was simple. Simple won.

Nested browser agents were sophisticated. Direct tool attachment was simple. Simple won.

Manual OTEL configuration was sophisticated. Letting AgentCore auto-inject it was simple. Simple won.

Every time we chose the more complex option, we eventually replaced it with the simpler one. The complex option wasn't wrong — it just wasn't necessary yet. Start simple. Add complexity only when you can prove the simple version is insufficient.

---

*This is Part 6 of a 6-part series. [← Part 5: Browser UI Testing](05-browser-ui-testing.md)*

---

## Full Series

| Part | Theme |
|------|-------|
| 1 | The idea + why prompt-only guardrails fail |
| 2 | Observability reveals token economics + auth problems |
| 3 | Sequential plan replaces worker complexity |
| 4 | Remote deployment on AgentCore |
| 5 | Browser UI testing + eliminating nested agents |
| 6 | Production patterns (this article) |
