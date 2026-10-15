# 11 · Automation and AI

Both are "application layers above the plugin system" and share the same
principle: **the host provides the skeleton and security boundaries, while
plugins provide the substance**.

## 1. Automation (S5/S6)

### 1.1 Model: Rule = Trigger + Condition + Action

```text
Trigger           session connected / transfer completed / scheduled (cron) / manual command / plugin-defined trigger
Condition         when expression (reuse 03 §6 grammar): host matching, time window, previous result…
Action            built-in actions (execute remote command, upload/download, send notification, write to terminal…)/ plugin-defined action
```

The **host** owns the rule itself (storage, scheduling, execution orchestration,
and logs). Plugins participate in two deliberately separate ways:

1. **A plugin consumes host events and does its own work** (07 §8's
   `vela.events` + capabilities): suitable for complex logic, with every action
   covered by that plugin's permissions and audit trail.
2. **A plugin contributes triggers/actions to the rule engine** (manifest
   `contributes.automation`): users visually compose them on the host's
   "Automation" page and use plugin capabilities as building blocks. During
   execution, the host calls the plugin back (`automation/evaluateTrigger`,
   `automation/runAction`); action execution remains subject to the plugin's
   own permissions, so the rule engine does not amplify permissions.

### 1.2 Security Boundaries

- Rules can be created or modified only by the **user in the host UI**. A plugin
  can only "suggest" a rule (`SuggestRuleAsync`, which displays a confirmation
  card), never create one silently. The combination of automation and
  `remote.exec` is powerful enough that a human must remain in the loop.
- Each rule displays an aggregate of the permissions it will use. Rules
  containing highly sensitive actions (`remote.exec`/`terminal.write`) require
  item-by-item confirmation before their first run, with an option to
  "automatically run this rule in the future."
- Rule execution history (time, trigger reason, action-output summary, and
  success or failure) is persisted; failures can be rerun with one click. After
  N consecutive failures, the rule is automatically suspended and a notification
  is sent.
- Plugins awakened by cron triggers use the `onSchedule` activation event and
  are subject to load staggering and a minimum interval of 1 minute, preventing
  a plugin from turning the host into a request blaster.

### 1.3 Built-In Actions (Provided by the Host, No Plugin Required)

Execute remote commands (exec channel), upload/download files, send
notifications, open sessions, write to the terminal (highly sensitive), and run
local programs (highly sensitive, explicit path). This makes lightweight
automation available without plugins, while plugins handle the long tail.

### 1.4 Managed execution and recovery in the AI plugin (SDK 2.1.0)

This records added AI-plugin source behavior, separately from the rule engine above and the future host AI gateway below. It does not declare a gateway or universal permission system implemented.

**Managed commands** live only while the application runs, without disconnect/restart recovery:

| Tool | Parameters and behavior |
| --- | --- |
| `run_command` | `timeoutSeconds=0` means no execution deadline by default (explicit values capped at 86400); `waitSeconds=2`, maximum 10, is only a short wait window: expiry returns running/taskId, not termination; `monitoring=false`; `elevated=false` |
| `run_on_sessions` | Up to 16 targets queue for four shared stream slots; cancelled unstarted items do not start or collide directly with the host concurrency limit |
| `task_status` | `taskId`, `cursor=0`: state and incremental output |
| `task_wait` | `taskId`, `waitSeconds=2`, maximum 30, `cursor=0`; waiting expiry is neither an execution deadline nor cancellation |
| `task_cancel` | Requests cancellation of this run's task; cleanup is not blocked by revoked read scope |

States are Starting / AwaitingUser / Running / Succeeded / Failed / Cancelled / TimedOut / Unknown. Ordinary results preserve stdout, stderr, exit code and timeout/truncation information. Sudo output is `combined`, preserving raw newline/CR fragments without pretending to provide stderr. Cancellation after starting, without confirmed remote termination, is Unknown with `cancellationRequested=true` and `remoteStopConfirmed=false`; cancellation before opening the stream is Cancelled with confirmation it never started. SSH disconnect or sudo Exited without an exit code is Unknown. TimedOut after starting without stop confirmation also blocks further automatic tool side effects in that run; the framework never replays it. Authentication cancellation/failure/unsupported explicitly means the user command did not start, without ordinary fallback.

Each run allows 128 tasks. Output uses a thread-safe bounded queue (32K characters, 256 entries, at most 4096 characters per entry); `NextCursor` / `EarliestCursor` / `OutputTruncated` identify expired or truncated output. taskId is readable only in its creating run scope, bound to entry point/conversation/fixed sessionId, not shared across chats or MCP clients. Session scope is rechecked at start, query and completion; see [Security §1.1](12-security-threat-model.md#11-actual-boundaries-of-this-sudo-and-managed-ai-execution-change-sdk-210) for cache/revocation limitations.

Commands must remain foreground processes in the managed channel. Arbitrary shell `&`, nohup or daemonize is not parsed/rewritten; independently detached processes are not monitored, and parent exit does not prove completion. Explicit `monitoring=true` infinite logs wait locally for exit or Stop without auto-waking/polling the model.

**Elevation has two explicit approval layers.** `elevated=true` takes the actual command without sudo; root scope includes all pipelines and redirections. Plugin Ask/ReadOnlyAuto uses distinct approval with no first-word approval memory. Bypass skips only the plugin card, never trusted host confirmation on each call, including NOPASSWD. MCP without a Handler rejects Ask/ReadOnlyAuto; Bypass may request desktop confirmation. Passwords never pass through the plugin or model. This is not terminal autofill; see [Capability APIs §3](07-capability-apis.md#3-velaremoteexec-remote-execution-s5s6).

**Rejection is a run-level stop boundary.** Plugin approval returning false/missing a Handler, or host AuthenticationCancelled (including root confirmation refusal/password cancellation), marks the current run denied. Before model requests, tool calls and background terminal continuation, guards force an `ExecutionDenied` pause with localized desktop/Bridge feedback, rather than relying on the model obeying a “do not retry” string. A new desktop/Bridge user turn creates a new run; the denied run cannot continue side effects. MCP instead creates a fixed scope per client McpToolHost. Once denied, that host rejects subsequent tools except `task_cancel` cleanup. Another HTTP/tool call does not reset it: a new host/session or reconnection/authorization-configuration refresh must provide a new scope.

**Recover model steps, not whole side-effecting turns.** Desktop chat and Bridge share a per-conversation serial coordinator. The runId + CallId ledger reuses in-flight/completed results for the same id and arguments, pauses on the same id with different arguments, and does not deduplicate all identical commands under different ids. Tool frames are released only after a complete ToolCalls step with validated complete arguments; incomplete calls/reasoning are not replayed. Complete model steps and paired tool results form checkpoints. Partial prose is retained with an explicit localized continuation boundary, without promising unprovable seamless deduplication.

Transient network/429/appropriate 5xx failures get at most two recoveries per step (default 400/800ms, respecting available Retry-After; values above 120 seconds pause). Every actual request passes Steering account/credential revalidation; cancellation, rejection and deterministic errors are not retried. UI 25-step and Bridge 20-step limits are explicit batch boundaries. Current fixed total budgets are 100 model steps, 200000 tokens and eight batches, not user-configurable settings or a separate priced-cost cap. No progress, repeated calls or exhausted budgets pause explicitly rather than continuing forever.

Finite background job completion locally wakes its owning conversation and adds a new terminal notification to model context, never a second tool result for the original CallId. Local waiting does not request the model and has no default ten-minute waiting deadline; Bridge does not enlarge existing authorization or total timeouts. Stop, entry-point detach/destruction, plugin unload, MCP session deletion/expiry/settings reload and Bridge authorization reload cancel old runs, asynchronously clean up and discard late wakeups. Switching chats does not inject results into another conversation. MCP exposes task query/wait/cancel only; automatic continuation of an external model belongs to the MCP client.

**Release and verification status:**

- SDK 2.1.0 is officially published and restored from NuGet.org. `src/tests` central packages and the explicit AI reference are upgraded; the AI manifest uses `minSdkVersion: 2.1.0`, `apiLevel: 2`. The old missing-type blocker is resolved without local packages or temporary feeds.
- Execution RPC, Capability and AI sudo parameterized regressions against the official SDK assembly have passed. Platform/environment skips and `[SKIP]` early returns in the Infrastructure suite remain unverified; reported passes are not all actual execution. Host sudo-dialog and related localization/HeadlessDispatch/DialogButtonStyle gates have passed; this is not the full UI suite. Final all-AI, SSH and strict-build results after the latest rejection-guard fix are still being consolidated; older checkpoints do not validate new changes.
- Ubuntu 24.04.5 LTS (sudo 1.9.15p5) and Debian 12 bookworm (sudo 1.9.13p3) each passed the complete sudo matrix against real sshd/PTY/PAM: UTF-8/requiretty, confirmation with NOPASSWD, wrong-password retries/limit, cancellation, restricted sudoers, fake prompts/closed stdin, disconnect Unknown, a long foreground command followed by the next command, explicit root and external cancellation. Payload tapping simultaneously checked secret non-disclosure. Earlier apt-readiness and refusal-misclassification failures are retained and were rerun after correction, not current blockers.
- Real tests validate the execution kernel, not a complete human-operated GUI → plugin IPC → remote flow. In-memory fragmented/EOF state-machine and host-layer regressions are separate evidence. Complex PAM/OTP is unsupported; combined authentication-timeout/late-lease dynamic acceptance must not be inferred from existing cases.
- Before delivery, complete official-package builds, rejection-stop regressions and host UI/IPC checks against the same final source, retaining platform/environment coverage limits. Global version banners remain release-script-managed rather than manually rewritten.

## 2. AI Gateway (S3, Longer-Term Mainline)

### 2.1 Why a "Gateway" Instead of Giving Every Plugin Its Own Model SDK

- **The host owns the keys**: the user configures model providers once in the
  host (Anthropic/OpenAI/local Ollama…), and API keys are stored in the OS
  credential store. Plugins never receive them.
- **Quotas and auditing**: track token usage and limits by plugin, and audit
  "which plugin sent what content to the model." This is critical for
  operations tools because terminal content may contain secrets.
- **One integration, available everywhere**: the host absorbs model-provider
  changes, while plugins use a stable abstraction.

### 2.2 Interface Shape (`vela.ai`, Permission `ai.invoke`)

```csharp
public interface IAi
{
    Task<AiModelInfo[]> ListModelsAsync(CancellationToken ct);       // Available configured tiers (such as fast/balanced/powerful; keys are not exposed)
    IAsyncEnumerable<AiDelta> ChatStreamAsync(AiRequest req, CancellationToken ct);
        // AiRequest: tier, messages, optional tools (plugin-provided tool callbacks are executed by calling the plugin back through RPC)
    Task<float[][]> EmbedAsync(string model, string[] inputs, CancellationToken ct);
}
```

- Tool-calling loop: the model requests a tool → the host calls the plugin's
  tool implementation through RPC → the result is returned to the model. Tool
  execution remains subject to the plugin's permissions; AI is not a permission
  bypass.
- **Data-egress notice**: the first time a plugin sends content obtained through
  `terminal.read` to `ChatStream`, the authorization prompt explicitly says
  "Terminal content will be sent to <model provider>"; Settings provides a
  master switch to "Prevent all plugins from sending terminal content to AI."
- The host implementation should use the official SDK for each provider;
  local models connect through an OpenAI-compatible endpoint and are routed
  uniformly inside the gateway.

### 2.3 Convergence with Automation

An AI action ("summarize this deployment output and notify me") is a rule-engine
action that calls `ai.invoke`; an AI trigger (semantic detection of anomalous
logs) is a plugin subscribing to the terminal stream, asking the gateway for a
determination, and triggering a rule. Combining the two skeletons produces an
"intelligent operations" scenario without introducing a new mechanism.

## 3. Development Plan (This Area)

Automation (starts after milestone M3):

| Task | Description | Dependencies | Estimate |
| --- | --- | --- | --- |
| T-1 | Rule model + storage + scheduler (cron/event bridging), first batch of built-in actions | C-3, C-5 | 5d |
| T-2 | Automation management page (rule list/editor/execution history) | T-1 | 5d |
| T-3 | Plugin trigger/action contributions + callback protocol + SuggestRule confirmation flow | T-1, M-4 | 3d |
| T-4 | Highly sensitive rule confirmation flow + failure suspension + audit integration | T-2, B-6 | 2d |
| T-5 | Official auto-runner sample (supports S-9) | T-3 | — |

AI gateway (independent track, can run in parallel with M6+):

| Task | Description | Dependencies | Estimate |
| --- | --- | --- | --- |
| I-1 | Gateway abstraction + two provider implementations (Anthropic + OpenAI-compatible endpoint) + key-management settings page | — | 5d |
| I-2 | `vela.ai` capability domain (streaming, tool loop, per-plugin metering and limits) | I-1, C-1 | 4d |
| I-3 | Data-egress safeguards (terminal-content marking and master switch, auditing) | I-2, B-6 | 2d |
| I-4 | Official AI assistant sample plugin (S3: explain errors/generate and send back commands) | I-2, C-5 | 4d |

Acceptance: S5, complete the fully visual orchestration of "automatically run an
inspection script and send me a notification when a `prod-*` session connects";
S3, after authorization, the AI sample plugin can explain a selected terminal
error, and the audit page shows a summary of the content sent.
