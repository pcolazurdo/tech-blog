# Deploying to the Cloud — AgentCore Runtime

*Part 4 of 6: Taking a local CLI tool to a remote cloud service*

---

At the end of Part 3, we had a working testing platform: a CLI that coordinated five AI agents to analyze a system, generate a test plan, and execute tests one at a time. It worked great on my laptop. But a CLI tool that only runs locally has a shelf life — you can't share it with a team, you can't run it from a CI pipeline, and you definitely can't put a React UI in front of it.

This article covers the deployment to AWS Bedrock AgentCore — turning the local multi-agent system into a cloud service that any client can invoke over HTTPS.

## Why AgentCore?

The platform uses Strands SDK agents backed by Bedrock models. AgentCore is the natural deployment target because:

1. **Session affinity** — it routes the same `runtimeSessionId` to the same microVM. Our multi-turn conversation (with PlanStore state in memory) survives across invocations without any serialization.
2. **No infrastructure to manage** — no ECS clusters, no Lambda cold starts to tune. It's a container that runs when invoked.
3. **Built-in auth** — SigV4 signing handles authentication. No API keys to rotate.

The microVM stays alive for up to 8 hours and idles for 15 minutes. In practice, a typical testing session (analyze → plan → execute 10 tests → done) takes 15-30 minutes. The microVM lives for exactly as long as needed.

## The Architecture Split

The key decision: what stays local and what moves remote?

```
LOCAL (CLI)                          REMOTE (AgentCore microVM)
─────────────                        ────────────────────────────
- boto3 SigV4 auth                   - All 5 agents
- Session ID lifecycle               - PlanStore singleton
- Upload config files to S3          - Browser tools (UI mode)
- Display responses (Rich)           - Skills (bundled in container)
- Download artifacts from S3         - Target system HTTP calls
```

The CLI becomes a thin client. It doesn't create agents, doesn't import Strands, doesn't need Bedrock credentials beyond the ability to invoke the runtime. All the heavy lifting happens in the microVM.

## Config Delivery: The S3 Pattern

The first design had the CLI sending everything in the invocation payload:

```python
payload = {
    "prompt": "test leave workflow",
    "config": {...},  # agents.yaml contents
    "openapi_spec": "... 2MB of YAML ...",
    "target_api_key": "secret",
}
```

This works (AgentCore supports 100MB payloads) but has problems:
- Secrets flow through the invocation API
- The payload is logged in CloudTrail
- Large specs make each invocation slower

The fix: **upload files to S3 first, send only the S3 prefix in the payload.**

```python
# CLI: upload before first invocation
s3.upload_file("config/agents.yaml", bucket, f"sessions/{sid}/init/agents.yaml")
s3.upload_file("openapi.yaml", bucket, f"sessions/{sid}/init/openapi.yaml")
s3.upload_file(".env", bucket, f"sessions/{sid}/init/.env")

# First invocation payload — just the pointer
payload = {
    "prompt": "test leave workflow",
    "init": True,
    "config_s3_prefix": f"s3://{bucket}/sessions/{sid}/init/",
}
```

The entrypoint downloads everything to `/tmp/session/` on initialization. Subsequent invocations just send `{"prompt": "continue"}`.

## The Entrypoint

The AgentCore entrypoint is surprisingly simple — because all the agent wiring is in the shared `agent_factory.py`:

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()
_orchestrator = None

@app.entrypoint
def invoke(payload: dict) -> dict:
    global _orchestrator

    if payload.get("init") or _orchestrator is None:
        _download_session_files(payload["config_s3_prefix"])
        config = _build_config()
        _orchestrator = _create_orchestrator_from_config(config)

    output_mode = payload.get("output_mode")
    if output_mode:
        _session_config.execution.output_mode = output_mode

    result = _orchestrator(payload["prompt"])
    return {
        "response": str(result),
        "metrics": _extract_metrics(result),
        "plan_progress": _get_plan_progress(),
    }
```

The `_orchestrator` is a module-level variable that persists across invocations. AgentCore's microVM affinity means the same process handles all invocations for a session. The PlanStore singleton, the agent's conversation history (`agent.messages`), everything stays in memory.

The `output_mode` override lets a single deployed container image serve multiple execution modes (API, CLI-style, UI) without rebuilding or redeploying — the caller specifies the mode in each invocation payload.

## The Agent Factory: One Code Path, Two Runtimes

The same `create_all_agents()` function is used by both the CLI and the remote entrypoint:

```python
def create_all_agents(config, callback_handler=None, session_id=None, ...):
    config_analyzer = create_config_analyzer(...)
    test_planner = create_test_planner(...)
    test_executor = create_test_executor(...)
    failure_analyst = create_failure_analyst(...)
    orchestrator = create_orchestrator(
        config_analyzer=config_analyzer,
        test_planner=test_planner,
        test_executor=test_executor,
        failure_analyst=failure_analyst,
        ...
    )
    return {"orchestrator": orchestrator, ...}
```

The only difference: the CLI passes a `_TeeCallbackHandler` (for session log tee-ing), while the remote entrypoint relies on CloudWatch Logs (stdout goes to CloudWatch automatically via the execution role).

## Container Packaging

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /build
COPY pyproject.toml src/ skills/ .
RUN pip install --no-cache-dir .

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages/ ...
COPY src/ /app/src/
COPY skills/ /app/skills/
ENV PYTHONPATH=/app PYTHONUNBUFFERED=1
ENTRYPOINT ["opentelemetry-instrument", "python", "-m", "src.runtime.entrypoint"]
```

Key details:
- **ARM64** — AgentCore requires it
- **Skills bundled at `/app/skills/`** — the `build_skills_plugin()` function resolves paths relative to the working directory
- **`opentelemetry-instrument` prefix** — AgentCore injects all ADOT env vars automatically before the process starts; no manual OTEL configuration needed

> **🔑 Lesson Learned: Don't Fight the Platform**
>
> Our first attempt manually configured OTEL env vars in the Dockerfile and CDK stack — `AGENT_OBSERVABILITY_ENABLED`, `OTEL_PYTHON_DISTRO`, exporter settings. These conflicted with the values AgentCore auto-injects before the process starts. The result: every span export returned HTTP 400.
>
> The fix was three lines deleted, zero added. `opentelemetry-instrument` as the entrypoint is correct — but you must let AgentCore configure it. When a managed runtime handles something for you, get out of the way.

## Configurable Log Levels

The entrypoint reads `LOG_LEVEL` from the environment:

```python
_log_level = getattr(logging, os.environ.get("LOG_LEVEL", "INFO").upper(), logging.INFO)
logging.basicConfig(level=_log_level, ...)
logging.getLogger("src").setLevel(_log_level)
logging.getLogger("strands").setLevel(logging.WARNING)
logging.getLogger("botocore").setLevel(logging.WARNING)
```

Production runs at INFO. Debug runs at DEBUG. Noisy libraries are always at WARNING. Same pattern as the CLI's `--debug` flag, but via environment variable.

## The CDK Stack

The infrastructure is minimal:

```python
class AgentCoreStack(Stack):
    # S3 artifacts bucket (session files + results)
    artifacts_bucket = s3.Bucket(..., block_public_access=BLOCK_ALL)

    # IAM execution role for the runtime
    execution_role = iam.Role(
        assumed_by=ServicePrincipal("bedrock-agentcore.amazonaws.com"),
        inline_policies={
            "bedrock": PolicyStatement(actions=["bedrock:InvokeModel"], resources=["*"]),
            "s3": PolicyStatement(actions=["s3:GetObject", "s3:PutObject"], resources=[...]),
            "logs": PolicyStatement(actions=["logs:*"], resources=["*"]),
            "aoss": PolicyStatement(actions=["aoss:APIAccessAll"], resources=[...]),
            "browser": PolicyStatement(
                actions=["bedrock-agentcore:*BrowserSession*"],
                resources=["*"]
            ),
        },
        environment={
            "AWS_BROWSER_REGION": "us-east-1",
            "BROWSER_IDENTIFIER": browser_identifier,
        }
    )
```

## Multi-Turn Mapping

The CLI's `while True` loop maps cleanly to AgentCore invocations:

| Turn | CLI Sends | AgentCore Does |
|------|-----------|----------------|
| 1 | `{init: true, prompt: "test leave", config_s3_prefix: "s3://..."}` | Initialize agents, run ANALYZING | 
| 2 | `{prompt: "yes, generate plan"}` | Run PLANNING |
| 3 | `{prompt: "approve, start executing"}` | Execute test 1 |
| 4 | `{prompt: "continue"}` | Execute test 2 |
| N | `{prompt: "quit"}` | Upload final artifacts |

Each invocation is one turn. The orchestrator's one-phase-per-turn design maps perfectly — it was designed for this from the start.

## Artifact Upload

After each invocation the entrypoint uploads session artifacts to S3 under `s3://{bucket}/sessions/{session_id}/`:

| Path | Content |
|------|---------|
| `progress.json` | PlanStore progress snapshot |
| `test_plan.json` | Full generated test plan |
| `scripts/` | Generated test scripts |
| `results/` | Per-test execution results |

This happens on every return, not just at session end. If the microVM is recycled or the client disconnects, the last uploaded snapshot is still there. It enables post-session analysis, audit trails, and the CLI's artifact download step that surfaces results to the user.

## What I'd Do Differently

1. **Start with S3 config delivery.** Embedding config in the payload was a dead end. Files-on-S3 is the right pattern for anything over 1KB.
2. **Don't override platform-managed configuration.** AgentCore auto-injects OTEL env vars before the process starts. Our manual overrides (`AGENT_OBSERVABILITY_ENABLED`, `OTEL_PYTHON_DISTRO`, exporter settings) caused HTTP 400 errors on every span export. The fix was deletion, not addition.
3. **Make the agent factory the first abstraction.** Having `create_all_agents()` shared between CLI and entrypoint meant the remote deployment was a one-day task, not a one-week refactor.

---

*This is Part 4 of a 6-part series. [← Part 3: Sequential Plan](03-sequential-plan-architecture.md) | [Part 5: Browser UI Testing →](05-browser-ui-testing.md)*
