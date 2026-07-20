# Blog Series 2: From Local CLI to Production Cloud Service

## Series recap

### What Series 1 covers (Parts 1-3)

| Part | Title | Key Theme |
|------|-------|-----------|
| 1 | Building an AI-Powered Testing Agent | The idea, initial multi-agent architecture, why prompt guardrails fail |
| 2 | The V2 Architecture | Observability reveals token economics, auth isolation, WorkItem fragility |
| 3 | The Sequential Plan Architecture | Removing workers, PlanStore, one-test-per-turn, simplicity wins |

### What Series 2 covers (Parts 4-6)

| Part | Title | Key Theme |
|------|-------|-----------|
| 4 | Deploying to the Cloud — AgentCore Runtime | Containerized agent as a cloud service with session affinity |
| 5 | Adding Eyes — Browser-Based UI Testing | From API-only to visual testing; eliminating the sub-agent anti-pattern |
| 6 | Patterns for Production Agent Systems | Cross-cutting operational lessons from all three execution modes |

---

## Part 4: Deploying to the Cloud — AgentCore Runtime

**Theme:** Taking a local CLI tool and deploying it as a remote cloud service on AWS Bedrock AgentCore.

**Opening hook:** "It worked great on my laptop. But a CLI tool that only runs locally has a shelf life — you can't share it with a team, you can't run it from a CI pipeline, and you can't put a React UI in front of it."

**Key topics:**

1. **Why AgentCore over ECS/Lambda:**
   - Session affinity (same `runtimeSessionId` → same microVM) — PlanStore singleton survives
   - No infrastructure to manage — no clusters, no cold-start tuning
   - Built-in SigV4 auth — no API keys to rotate
   - 8-hour microVM lifetime, 15-min idle timeout

2. **The architecture split:**
   - Local CLI (`src/cli/remote.py`): boto3 SigV4 auth, session ID lifecycle, S3 config upload, streaming response handler
   - Remote container (`src/runtime/entrypoint.py`): all agents, PlanStore, browser tools
   - `src/runtime/agent_factory.py`: shared agent construction between local and remote paths

3. **S3-based config delivery:**
   - Problem: agents.yaml + .env + openapi.yaml exceed AgentCore payload limits
   - Solution: CLI uploads to `s3://{bucket}/sessions/{session_id}/init/`, entrypoint downloads to `/tmp/session/`
   - .env loaded via python-dotenv, API key extracted to env var

4. **Container packaging:**
   - ARM64, Python 3.12, multi-stage build
   - Local packages copied: `COPY packages/ packages/` for strands-browser-agent
   - ENTRYPOINT: `opentelemetry-instrument python -m src.runtime.entrypoint`
   - No OTEL env var overrides — AgentCore auto-injects all ADOT config

5. **Multi-turn conversation mapping:**
   - Each CLI prompt → one AgentCore `invoke_agent()` call
   - First call includes `init: true` + `config_s3_prefix`
   - Subsequent calls send only `prompt` — agents already initialized on the microVM
   - Payload override: `output_mode` can switch execution mode per-invocation

6. **CDK infrastructure (`cdk/stacks/agentcore_stack.py`):**
   - Runtime env vars: ARTIFACTS_BUCKET, KB_KNOWLEDGE_BASE_ID, KB_REGION, AWS_BROWSER_REGION, BROWSER_IDENTIFIER
   - IAM: bedrock:InvokeModel (including inference-profile/*), s3, logs, xray, aoss, bedrock-agentcore:*BrowserSession*
   - ECR image auto-deployed

7. **Artifact upload after each turn:**
   - PlanStore progress, test scripts, and results uploaded to `s3://{bucket}/sessions/{session_id}/`
   - Enables post-session analysis and audit trail

**Lesson Learned box:** "Don't fight the platform. We spent two days debugging OTEL 400 errors caused by manually setting env vars that AgentCore auto-injects. The fix was to *remove* our configuration — three lines deleted, zero added. When a managed runtime handles something for you, let it."

**Code snippets to include:**
- `_download_session_files()` showing the S3 paginator pattern
- The `invoke()` entrypoint showing init vs subsequent flow
- CLI streaming response handler
- CDK IAM policy with inference-profile wildcard

**Design doc references:** `docs/design/remote-agentcore-architecture.md`, `docs/design/04-agent-model-configuration.md`

---

## Part 5: Adding Eyes — Browser-Based UI Testing

**Theme:** Extending from API-only testing to visual/interactive testing via a remote Chrome browser. The story of how a simple "add a browser tool" request turned into a fundamental architecture lesson.

**Opening hook:** "API testing proves the backend works. But enterprise SaaS users don't interact with APIs — they interact with forms, buttons, and dashboards. A configuration that passes every API test can still fail in the UI."

**Key topics:**

1. **The three execution modes:**
   - `execute` → TestExecutor generates httpx Python scripts, runs in subprocess
   - `nl` → exports plan as structured markdown (no execution)
   - `ui` → UITestExecutor drives remote Chrome via CDP
   - Single orchestrator, mode-specific tool wiring via `output_mode`

2. **AgentCore Browser:**
   - Managed Chrome instances in AWS
   - CDP endpoint for Playwright automation
   - DCV live view stream for real-time observation
   - `LiveViewBrowser` subclass in `packages/strands-browser-agent/`

3. **The architecture mistake (two-agent anti-pattern):**
   ```
   BEFORE: orchestrator → execute_ui_test tool → browse_website tool → internal Agent → browser
   ```
   - The internal agent knew HOW to drive a browser but not WHY
   - It lacked: auth credentials, test assertions, base URL context
   - Two LLM calls, each with less information than one would have

4. **The fix (direct browser tool):**
   ```
   AFTER: orchestrator → execute_ui_test tool → UITestExecutor agent → browser tool (directly)
   ```
   - One agent, full context, one reasoning loop
   - UITestExecutor system prompt includes: session_name, base_url, auth credentials, browser action syntax
   - Browser tool passed directly: `tools=[browser_tool]`

5. **Authentication handling:**
   - Credentials from UI_USERNAME / UI_PASSWORD env vars (never in logs or state)
   - System prompt includes auth instructions: detect login page → fill → submit → verify
   - ValueError raised at agent creation if credentials missing

6. **Session management:**
   - `shared`: one browser reused across tests (faster, state bleeds)
   - `isolated`: fresh browser per test (slower, clean state)
   - `FileSessionManager` preserves conversation history across multi-step tests
   - Browser created lazily on first `execute_ui_test` call (avoids startup cost if cancelled)

7. **Monitor server integration:**
   - `active_sessions` dict tracks live browser sessions
   - HTTP polling for DCV view URLs
   - `close_browser` tool terminates session and releases resources

8. **Tool wiring in orchestrator:**
   - UI mode adds: `execute_ui_test`, `close_browser`, `mark_test_complete`
   - UI mode removes: `execute_tests`, `analyze_failures`
   - System prompt appends `UI_MODE_PROMPT_SUFFIX` with UI-specific workflow phases

**Lesson Learned box:** "When you have Agent A calling Tool X which creates Agent B, you've created a reasoning gap. Agent B doesn't know what Agent A knows. The fix isn't to pass more context — it's to eliminate Agent B entirely and let Agent A use the tool directly. One well-informed agent beats two partially-informed agents every time."

**Code snippets to include:**
- `create_ui_test_executor()` showing direct browser tool attachment
- System prompt template with session_name and auth section
- Orchestrator `output_mode == "ui"` branch showing tool wiring
- Browser action examples (navigate, fill, click, screenshot)

**Design doc references:** `docs/design/05-execution-modes-and-output.md`, `docs/design/browser-ui-testing.md`, `docs/design/ui-executor-direct-browser.md`

---

## Part 6: Patterns for Production Agent Systems

**Theme:** Cross-cutting lessons from deploying and operating this system across local, remote, and browser modes. Not theoretical best practices — hard-won operational knowledge from watching agents fail.

**Opening hook:** "After six months of building, breaking, and rebuilding this platform — across three execution modes, two deployment targets, and dozens of debugging sessions — patterns emerged. All cost time to learn the first time."

**Key topics (patterns):**

1. **Defense in Depth for Agent Behavior:**
   - Layer 1: Prompt instructions (aspirational — LLMs may ignore)
   - Layer 2: Tool-level enforcement (code rejects invalid actions, e.g., `allow_write_methods=False`)
   - Layer 3: Architectural constraints (one-phase-per-turn physically prevents chaining)
   - Examples from each layer across all three modes

2. **One Well-Informed Agent > Two Partially-Informed Agents:**
   - The browser sub-agent anti-pattern (proven failure)
   - Exception: different model tiers (cheap model for bulk classification)
   - Rule: if both agents use the same model, eliminate the intermediary

3. **Observability Before Architecture:**
   - Per-agent token metrics: `[Metrics] agent=config_analyzer tokens={input: 273612, ...}`
   - Session-scoped debug logs: `.state/debug-{session_id}.log`
   - OTEL spans on PlanStore: test lifecycle visibility
   - HTTP request log: every API call timestamped
   - Table of real incidents + time-to-diagnose with tracing vs without

4. **The PlanStore Pattern (In-Memory State with Tracing):**
   - Singleton shared across all tools without passing references
   - No serialization overhead, survives across turns (same process/microVM)
   - OTEL-instrumented: every state transition emits a span
   - When to use: session-scoped state where durability isn't critical

5. **Skills for Domain Specialization:**
   - `skills/general/` for shared knowledge, `skills/{agent_name}/` for specifics
   - SKILL.md marker file → content injected via AgentSkills plugin
   - Lazy-loaded, graceful degradation if plugin unavailable
   - Zero-code extension: add a markdown file to change agent behavior

6. **Centralized Model Configuration (`bedrock_model_kwargs`):**
   - Problem: temperature/top_p hardcoded across 8 files, some models reject certain params
   - Solution: `AgentModelConfig.bedrock_model_kwargs()` — only sends non-None params
   - Principle: let the platform use its defaults unless you have a reason to override

7. **Security Hardening:**
   - `block_public_access` on all S3 buckets (caught by CDK review)
   - `trust_env=False` on HTTP clients (prevents .netrc auth injection)
   - Secrets via env vars only (never in config files committed to git)
   - Sample file pattern: `agents.yaml.sample` committed, `agents.yaml` gitignored
   - UI credentials validated at agent creation time (fail fast, not mid-test)

8. **CDK and Infrastructure Gotchas:**
   - AOSS policy propagation: 30-60s delay before KB can access collection
   - CfnJson for token resolution in AOSS access policies
   - Knowledge Base index creation: Lambda custom resource vs CDK (chose Lambda)
   - IAM: inference-profile/* wildcard needed for cross-region model invocation

9. **The Monitor Server Pattern:**
   - Sharing state between tools and a web frontend via in-process dict
   - HTTP polling for live browser session URLs
   - Useful for any tool that produces observable side effects

**Lesson Learned box:** "Every operational problem we hit was an observability problem first. Per-agent token metrics, session-scoped debug logs, and OTEL tracing on PlanStore turned 'it crashed' into 'config_analyzer consumed 273K tokens on a single call' in seconds. Add observability on day one. Not 'when we have time.' Day one."

**Code snippets to include:**
- `AgentModelConfig.bedrock_model_kwargs()` helper
- PlanStore singleton with OTEL spans
- `build_skills_plugin()` loading pattern
- Defense-in-depth layer comparison table

**Design doc references:** `docs/design/02-observability-and-telemetry.md`, `docs/design/03-skills-and-extensibility.md`, `docs/design/04-agent-model-configuration.md`, `docs/design/01-v1-to-sequential-plan.md`

---

## Series structure

| Part | Focus | Primary Audience | Estimated Length |
|------|-------|-----------------|-----------------|
| 4 | Remote deployment (AgentCore) | Engineers deploying agent systems to production | ~2500 words |
| 5 | Browser UI testing | Engineers building visual testing / RPA-adjacent systems | ~2500 words |
| 6 | Production patterns | Anyone operating multi-agent systems in production | ~3000 words |

## Relationship to existing series

Parts 1-3 tell the story of *building the core* (local, API-only, iterating on architecture).
Parts 4-6 tell the story of *going to production* (remote, multi-modal, operational patterns).

The two series are independent — a reader can start at Part 4 if they just want the deployment/operational lessons. Part 4 opens with a brief recap of where Part 3 left off.

## Cross-references to design docs

Each blog post should link to the corresponding design doc(s) for readers who want implementation-level detail:

| Blog Part | Design Docs |
|-----------|-------------|
| Part 4 | `remote-agentcore-architecture.md`, `04-agent-model-configuration.md` |
| Part 5 | `05-execution-modes-and-output.md`, `browser-ui-testing.md`, `ui-executor-direct-browser.md` |
| Part 6 | `02-observability-and-telemetry.md`, `03-skills-and-extensibility.md`, `04-agent-model-configuration.md`, `01-v1-to-sequential-plan.md` |

## Tone and style (same as series 1)

- Developer blog, Medium-style
- AWS services named directly (Bedrock, AgentCore, ADOT, X-Ray, S3, AOSS)
- Strands SDK specifics: Agent, @tool, BedrockModel, StrandsTelemetry, AgentSkills, FileSessionManager
- Code snippets (Python) and architecture diagrams (mermaid or ASCII)
- "Lesson Learned" callout boxes — one per article, distilling the single biggest takeaway
- Each article stands alone but links forward/backward
- Show failures before fixes: the journey matters more than the final state
- Concrete numbers: token counts, response times, lines of code changed
