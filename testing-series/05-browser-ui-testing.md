# Adding Eyes — Browser-Based UI Testing

*Part 5 of 6: From API scripts to a real Chrome browser*

---

API testing proves the backend works. But enterprise SaaS users don't interact with APIs — they interact with forms, buttons, and dashboards. A configuration that passes every API test can still fail in the UI: a dropdown missing an option, a button disabled by a CSS class, a form that submits but shows no confirmation.

This article covers adding a third execution mode — `ui` — that drives a real Chrome browser to execute tests visually. The journey from "just add a browser tool" to "completely rethink the agent architecture" is the core story.

## The Three Modes

After this feature, the platform supports three execution modes:

| Mode | Command | What Happens |
|------|---------|-------------|
| `execute` | `testing-agent run` | Test Executor generates Python scripts → subprocess |
| `nl` | `testing-agent run --output nl` | Export test plan as markdown document |
| `ui` | `testing-agent run --output ui` | UI Test Executor drives a remote Chrome browser |

The Config Analyzer and Test Planner work the same in all modes. Only the execution phase changes — and the Test Planner adapts its output format for UI-specific descriptions.

## AgentCore Browser: Remote Chrome in AWS

AWS provides AgentCore Browser — a managed service that runs Chrome instances in the cloud. You get:
- A CDP (Chrome DevTools Protocol) endpoint for Playwright automation
- A DCV live view stream for real-time monitoring (you can watch the browser)
- Managed lifecycle (session start, stop, timeouts)

The `strands-browser-agent` package wraps this as a Strands tool:

```python
from strands_browser_agent.browser import LiveViewBrowser

browser = LiveViewBrowser(region="us-west-2", identifier="aws.browser.v1")
# browser.browser is a Strands tool — the agent calls it like any other tool
```

## The First Architecture (Wrong)

My first implementation wrapped the browser in a `browse_website` tool with an internal sub-agent:

```
ui_test_executor Agent (has full context: auth, test steps, base_url)
  └── browse_website tool
        └── Internal Agent (generic "browser automation agent" prompt)
              └── browser_tool.browser (raw Playwright actions)
```

The `browse_website` tool created its own Strands Agent internally:

```python
@tool
def browse_website(goal: str, session_id: str = "") -> dict:
    agent = Agent(
        tools=[browser_tool.browser],
        system_prompt="You are a browser automation agent...",
    )
    response = agent(goal)
    return {"status": "success", "content": response}
```

This immediately failed in three ways.

### Failure 1: The internal agent didn't know the credentials

The ui_test_executor's system prompt said "Username: admin, Password: secret123". But the internal agent inside `browse_website` never saw that. It had a generic prompt: "Navigate pages and extract information." When it hit a login page, it had no idea what to do.

### Failure 2: The internal agent didn't know the test assertions

The test description said "Verify the page shows 'Application Approved'." But the internal agent only received a `goal` string — not the full test context with expected outcomes. It navigated successfully but couldn't determine pass/fail.

### Failure 3: The callback handler wasn't propagated

The session log showed the ui_test_executor's reasoning, then a gap (the internal agent's output went to stdout only), then the result. We fixed this with `set_browser_callback_handler()` — but it was treating the symptom, not the disease.

> **🔑 Lesson Learned: The Two-Agent Anti-Pattern**
>
> When Agent A calls a tool that creates Agent B, you've created a reasoning gap. Agent B doesn't know what Agent A knows. The fix isn't to pass more context to Agent B — it's to eliminate Agent B entirely and let Agent A use the tool directly.
>
> The internal sub-agent was paying for LLM reasoning twice: once in the ui_test_executor (which knew everything) and once in the browse_website agent (which knew nothing). One well-informed agent is always better than two partially-informed agents.

## The Fix: Direct Browser Tool

The rearchitecture is simple: give the `browser_tool.browser` directly to the ui_test_executor:

```
ui_test_executor Agent (has full context: auth, test steps, base_url)
  └── browser_tool.browser (raw Playwright actions — direct tool)
```

```python
def create_ui_test_executor(model_config, browser_tool, session_name, base_url, ...):
    system_prompt = f"""
    You are the UI Test Executor. You have DIRECT control of a Chrome browser.
    Session name: {session_name}
    Base URL: {base_url}
    Username: {ui_username}
    Password: {ui_password}

    Browser actions: navigate, click, fill, screenshot, get_text, wait, scroll
    Always include session_name='{session_name}' in every action.
    """

    return Agent(
        model=model,
        system_prompt=system_prompt,
        tools=[browser_tool],  # direct — no wrapper
    )
```

Now the agent that drives the browser:
- Knows the login credentials (in its system prompt)
- Knows the test assertions (in the prompt per test)
- Knows the session_name (baked into the system prompt)
- Has its output streamed through the callback handler (it's one agent, not two)

## Session Lifecycle: Managed by the Orchestrator

The browser session is created lazily — not when the ui_test_executor is constructed, but when the first test executes:

```python
def _ensure_session():
    instance = LiveViewBrowser(region=browser_region, identifier=identifier)
    session_name = f"session-{uuid4().hex[:8]}"

    # Initialize the browser session
    instance.browser(browser_input={
        "action": {"type": "init_session", "session_name": session_name}
    })

    # Register for monitoring
    active_sessions[session_id] = {
        "browser_tool": instance,
        "session_name": session_name,
        "live_view_url": instance.get_live_view_url(),
        "status": "ready",
    }
```

The orchestrator's `execute_ui_test` tool manages this lifecycle:
- **First test:** creates session, registers in `active_sessions`, creates ui_test_executor
- **Shared tests:** reuses the same session (cookies, login state preserved)
- **Isolated tests:** closes current session, creates a new one
- **Done:** `close_browser` tool stops the session

## Monitor Server: Live View

The monitor server (`monitor.py`) serves live view URLs to a frontend:

```python
@app.get("/sessions")
def list_sessions():
    return [
        {"session_id": sid, "live_view_url": state["live_view_url"], "status": state["status"]}
        for sid, state in active_sessions.items()
    ]
```

The CLI starts it automatically in UI mode:

```python
if output_mode == "ui":
    threading.Thread(target=_run_monitor, daemon=True).start()
    console.print("Browser monitor started on http://0.0.0.0:9900")
```

A future React UI can embed the DCV stream directly — real-time video of the browser as tests execute.

## Test Planner: UI Mode Suffix

When `output_mode == "ui"`, the Test Planner gets additional instructions:

```
--- UI MODE ---
Each test should describe:
1. NAVIGATION TARGET: URL path (e.g., "/app/leave/new")
2. UI ACTIONS: Click buttons, fill fields, select dropdowns
3. VISUAL VERIFICATIONS: Text present, element visible, form state

Use concrete labels: "Fill the 'Leave Type' dropdown with 'Casual Leave'"
Each test must specify session: "shared" or "isolated"
```

The output format includes a `session` field:

```json
{
  "id": "ui_test_01",
  "name": "submit_leave_happy_path",
  "description": "Navigate to /app/leave-application/new. Fill 'Leave Type' with 'Casual Leave'...",
  "category": "happy_path",
  "session": "shared"
}
```

## FileSessionManager: Memory Across Tests

The ui_test_executor uses `FileSessionManager` to persist its conversation history across multiple test executions:

```python
session_manager = FileSessionManager(
    session_id=session_id,
    storage_dir=".state/sessions",
)

return Agent(
    ...,
    session_manager=session_manager,
)
```

This means the agent remembers what happened in previous tests within the same session. If test 1 logs in, test 2 knows it's already logged in. If test 3 creates a record, test 4 can reference it (though we discourage test dependencies in the plan).

## What a UI Test Execution Looks Like

```
[Orchestrator] Executing UI test: ui_test_01 — valid_admin_login
  Browser session created: abc-123
  Live view: https://dcv-stream.amazonaws.com/...

[UI Test Executor]
  → browser(navigate to https://app.example.com/login)
  → browser(fill #username with "admin")
  → browser(fill #password with "***")
  → browser(click "Log in")
  → browser(screenshot)
  → Verifying: page shows dashboard...
  → Result: PASS — dashboard visible after login

[Orchestrator] Test ui_test_01: ✅ PASS
  Progress: 1/5 (pass: 1, fail: 0)
  Continue?
```

All browser actions appear as tool calls in the session log — full visibility.

## API vs UI: When to Use Which

| Aspect | API (`execute`) | UI (`ui`) |
|--------|----------------|-----------|
| Speed | 30-60s/test | 60-120s/test |
| Coverage | Backend logic, validations | Visual state, form behavior, CSS |
| Auth | API key/token | Browser login (username/password) |
| Evidence | HTTP traces (JSON) | Screenshots + live video |
| Flakiness | Low (deterministic) | Higher (animations, timing) |
| Cost | Bedrock tokens only | Bedrock + AgentCore Browser session |

Use API mode for validation rules, business logic, and regression. Use UI mode for visual verification, form behavior, and user journey testing. Use NL mode to generate the plan for a human or RPA tool.

---

*This is Part 5 of a 6-part series. [← Part 4: AgentCore Deployment](04-deploying-to-agentcore.md) | [Part 6: Production Patterns →](06-production-patterns.md)*
