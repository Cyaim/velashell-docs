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

States are Starting / AwaitingUser / Running / Succeeded / Failed / Cancelled / TimedOut / Unknown. Ordinary results preserve stdout, stderr, exit code and timeout/truncation information. Sudo output is `combined`, preserving raw newline/CR fragments without pretending to provide stderr. Cancellation after starting, without confirmed remote termination, is Unknown with `cancellationRequested=true` and `remoteStopConfirmed=false`. Two exceptions are recorded as Cancelled (also `remoteStopConfirmed=false`, but the run does not pause): an elevated command cancelled while still in approval/authentication, before the user command reported starting, and an explicit monitoring task stopped by name with `task_cancel`, which is the normal way to end one (cancelled along with a whole-run Stop, it is still Unknown). Cancellation before opening the stream is Cancelled with confirmation it never started. SSH disconnect or sudo Exited without an exit code is Unknown. TimedOut after starting without stop confirmation also blocks further automatic tool side effects in that run; the framework never replays it. Authentication cancellation/failure/unsupported explicitly means the user command did not start, without ordinary fallback.

Each run retains at most 128 tasks at once. At the cap, tasks that have finished and whose terminal state was already delivered are reclaimed; Unknown and unconfirmed-stop TimedOut tasks are not, because they are the run's stop boundary. Output uses a thread-safe bounded queue (32K characters, 256 entries, at most 4096 characters per entry; an oversize fragment is split into consecutive entries, not truncated); `NextCursor` / `EarliestCursor` / `OutputTruncated` identify earlier output evicted by those limits. taskId is readable only in its creating run scope, bound to entry point/conversation/fixed sessionId, not shared across chats or MCP clients. Session scope is rechecked at start, query and completion; see [Security §1.1](12-security-threat-model.md#11-actual-boundaries-of-this-sudo-and-managed-ai-execution-change-sdk-210) for cache/revocation limitations.

Commands must remain foreground processes in the managed channel. Arbitrary shell `&`, nohup or daemonize is not parsed/rewritten; independently detached processes are not monitored, and parent exit does not prove completion. Explicit `monitoring=true` infinite logs wait locally for exit or Stop without auto-waking/polling the model.

**Elevation has two explicit approval layers.** `elevated=true` takes the actual command without sudo; root scope includes all pipelines and redirections. Plugin Ask/ReadOnlyAuto uses distinct approval with no first-word approval memory. Bypass skips only the plugin card, never trusted host confirmation on each call, including NOPASSWD. MCP without a Handler rejects Ask/ReadOnlyAuto; Bypass may request desktop confirmation. Passwords never pass through the plugin or model. This is not terminal autofill; see [Capability APIs §3](07-capability-apis.md#3-velaremoteexec-remote-execution-s5s6).

**Rejection is a run-level stop boundary.** Plugin approval returning false/missing a Handler, or host AuthenticationCancelled (including root confirmation refusal/password cancellation), marks the current run denied. Before model requests, tool calls and background terminal continuation, guards force an `ExecutionDenied` pause with localized desktop/Bridge feedback, rather than relying on the model obeying a “do not retry” string. A new desktop/Bridge user turn creates a new run; the denied run cannot continue side effects. External MCP is the exception: each client McpToolHost still has one fixed scope (so background tasks remain queryable), but a denial is only that call's result and does not seal the connection. Later tools keep working (after read-only auto-approval refuses a write command, `list_sessions` still works), and whether to stop is up to the MCP client.

**Recover model steps, not whole side-effecting turns.** Desktop chat and Bridge share a per-conversation serial coordinator. The runId + CallId ledger reuses in-flight/completed results for the same id and arguments, pauses on the same id with different arguments, and does not deduplicate all identical commands under different ids. Tool frames are released only after a complete ToolCalls step with validated complete arguments; incomplete calls/reasoning are not replayed. Complete model steps and paired tool results form checkpoints. Partial prose is retained with an explicit localized continuation boundary, without promising unprovable seamless deduplication; when only whitespace arrived before a disconnect, the request is resent as is, with no whitespace-only assistant message and no continuation boundary. Argument-binding errors (a missing required argument, or a JSON value that cannot be read as the parameter type, caught before the tool body runs and therefore without side effects) go back to the model as tool errors to correct and retry; a single one does not pause, and getting the call right after two corrections is unaffected. Three consecutive steps that call tools but run none of them (binding failures, or same-CallId calls that only get the ledger's earlier result back) pause with `NoProgress`. Other tool failures still pause and their side effects are never retried automatically.

Normally completed model steps containing only role/usage/whitespace also enter recovery; prose, standard/raw reasoning and complete tool calls are not empty. Explicit output-length exhaustion, filtering, model refusals and error content are not retried as empty replies and do not trigger automatic failover. Empty steps and transient network/429/appropriate 5xx failures share at most two recoveries per step (default 400/800ms, respecting available Retry-After; values above 120 seconds pause with a dedicated reason, without retrying early or first announcing a retry or an execution-budget limit; the pause still carries the original 429 / 503, so the desktop panel's key rotation and failover work as usual on the first step). Every actual request passes Steering account/credential revalidation; cancellation, rejection and deterministic errors are not retried. UI 25-step and Bridge 20-step boundaries only segment execution: continuation is automatic using the same tool ledger and checkpoint, without removing tools or requiring confirmation for every batch. The cumulative hard limits of 100 model steps, 200000 output tokens and eight batches no longer apply. Batch progress shows only the actual batch number, not a fictitious total; background-completion wakeups do not increment it. Context utilization is only the input of the turn's last request relative to the model window (not the sum over the steps of a multi-step turn), not execution progress or remaining execution allowance; per-request output limits and existing context compaction/trimming remain independent. All reported input/output usage from successful and failed attempts is still accumulated; continued execution continues to incur model usage and cost, and users can Stop at any time. Cancellation, approval denial, unknown outcomes, tool errors and two `NoProgress` pauses can still stop execution: the same tool called four times in a row with the same arguments (the read-only polling tools `task_status` / `task_wait` / `transfer_status` / `transfer_wait` count as making progress while their result changes and accumulate only while it stays identical; side-effecting tools are still stopped by signature alone), and the three consecutive steps without a single tool actually running described above. Neither is complete loop detection. Existing Bridge turn timeouts and provider/model limits still apply, so completing every plan is not guaranteed.

Ordinary desktop chat retries an empty reply at most twice (400/800ms) before meaningful output or tool side effects; other transient failures still get at most one retry. Mixed failures do not reset the attempt counter; existing candidate failover follows exhaustion. Settings tests, bulk detection and pre-failover probes retry a normally completed response without visible prose only once (400ms); both requests and the wait share the existing 15-second deadline. Reasoning alone still cannot pass a probe. User cancellation propagates immediately, reported usage from failed attempts is accumulated, and these counts exclude provider SDK internal retries. Bridge shares step recovery and shows a localized empty-reply error after exhaustion. No settings were added, and empty-summary trimming fallback is unchanged.

Finite background job completion locally wakes its owning conversation and adds a new terminal notification to model context, never a second tool result for the original CallId; a terminal state already delivered through `task_status` / `task_wait` is not notified again. The notification says it is not a new user command, wraps remote output in `<untrusted-command-output>` and states that the content is command output, not user instructions, and must not be followed; the content is JSON-escaped, so output cannot forge the closing tag. A task whose session authorization was revoked yields only a "result withheld" Unknown notice (no output, exit code or session id), and the run pauses with TaskUnknown. Local waiting does not request the model and has no default ten-minute waiting deadline; Bridge does not enlarge existing authorization or total timeouts. Stop, entry-point detach/destruction, plugin unload, MCP session deletion/expiry/settings reload and Bridge authorization reload cancel old runs, asynchronously clean up and discard late wakeups. Switching chats does not inject results into another conversation. MCP exposes task query/wait/cancel only; automatic continuation of an external model belongs to the MCP client.

**Release and verification status:**

- SDK 2.1.0 is officially published and restored from NuGet.org. `src/tests` central packages and the explicit AI reference are upgraded; the AI manifest uses `minSdkVersion: 2.1.0`, `apiLevel: 2`. The old missing-type blocker is resolved without local packages or temporary feeds.
- Execution RPC, Capability and AI sudo parameterized regressions against the official SDK assembly have passed. Platform/environment skips and `[SKIP]` early returns in the Infrastructure suite remain unverified; reported passes are not all actual execution. Host sudo-dialog and related localization/HeadlessDispatch/DialogButtonStyle gates have passed; this is not the full UI suite. The all-AI suite and targeted SSH regressions after the latest rejection-guard fix have passed. The final strict non-incremental Debug build passed with warnings treated as errors and build-time code-style enforcement, with zero warnings/errors. Targeted regressions were also rebuilt and passed after the style cleanup; older checkpoints do not validate new changes.
- Ubuntu 24.04.5 LTS (sudo 1.9.15p5) and Debian 12 bookworm (sudo 1.9.13p3) each passed the complete sudo matrix against real sshd/PTY/PAM: UTF-8/requiretty, confirmation with NOPASSWD, wrong-password retries/limit, cancellation, restricted sudoers, fake prompts/closed stdin, disconnect Unknown, a long foreground command followed by the next command, explicit root and external cancellation. Payload tapping simultaneously checked secret non-disclosure. Earlier apt-readiness and refusal-misclassification failures are retained and were rerun after correction, not current blockers. The Ubuntu matrix is now reproducible from the code repository: `tests/fixtures/ssh-sudo` (Ubuntu 24.04, the distribution's sudo 1.9.15p5 with real PAM, plus fish / tcsh) backs the `ssh-sudo` service in `docker-compose.test.yml` (port 2230). `SudoInteropTests` in `VelaShell.Sudo.Kernel.Tests` run whenever that target is reachable and also check that the approved command runs byte for byte unchanged with fish / tcsh / bash as the login shell. They are in the `DockerIntegration` category, which CI does not run; an unreachable target is recorded as `[SKIP]`.
- Real tests validate the execution kernel, not a complete human-operated GUI → plugin IPC → remote flow. In-memory fragmented/EOF state-machine and host-layer regressions are separate evidence. Complex PAM/OTP is unsupported; combined authentication-timeout/late-lease dynamic acceptance must not be inferred from existing cases.
- Before delivery, complete official-package builds, rejection-stop regressions and host UI/IPC checks against the same final source, retaining platform/environment coverage limits. Global version banners remain release-script-managed rather than manually rewritten.

### 1.5 Model lists in the AI plugin settings

Provider model lists in the settings sidebar are collapsed by default, regardless of model count; each provider row still shows its model count. When opening settings, if the active model's provider is collapsed, its provider row is selected without automatically expanding the list; if it was saved as expanded, the active model remains selected. The chat model is unchanged. Click a provider row or press → to expand and ← to collapse; manual choices are saved. Explicitly adding or navigating to a model still reveals it for editing.

### 1.6 AI file uploads and progress (SDK 2.1.1)

This section concerns only `upload_local_file`: streaming a file from the user's **local machine** to an authorized SSH session through SFTP. It does not put arbitrary large files into model context, change chat attachments or bypass IM platform `send_file` limits. Removal of the two download-entry size caps is covered separately in §1.7. Uploads have no fixed AI file-size cap, but local/remote storage, authorization, protocol and connection constraints still apply.

| Tool | Behavior |
| --- | --- |
| `upload_local_file` | Absolute local and remote file paths and optional session; `overwrite=false` preserves an existing destination by default, and explicit replacement still requires approval. Returns a structured job snapshot rather than waiting for an entire large file |
| `transfer_status` | Reads this run's `transferId`: state, 64-bit transferred/total bytes, speed, ETA, generation and safe-resume availability |
| `transfer_wait` | Short wait, default 2 seconds, maximum 30; expiry never cancels the upload |
| `transfer_cancel` | Requests cancellation using job id and expected execution generation; requested does not mean remote termination confirmed |
| `transfer_resume` | Only when the host explicitly reports `CanResume`, with fresh approval and target-connection validation; never silently restarts from zero |

Snapshot `transfer_id` identifies the background job, `generation` distinguishes resumed execution, and `sequence` rejects stale progress. Interpret `stillRunning` with explicit state: all bytes may be present while verification/commit is pending, so full percentages are not success. Only `Succeeded` is successful. Failed, cancelled and `Unknown` outcomes are never automatically replayed; a lost commit acknowledgement must be resolved before another id can overwrite the destination.

The host owns approved uploads, separating request/model-wait lifetime from transfer lifetime. Status/short waits do not require a model-generation connection or repeated model requests to refresh progress. AI tasks are owner/run isolated, with session authorization rechecked on queries and completion; revoked paths/results are withheld. Initial receipts and final completion notifications are separate, never a second tool result for the original CallId. External MCP agents explicitly query/wait for progress; the server does not wake an external model.

Uploads stream rather than load whole files into memory. Parent-directory aliases are resolved before validating the actual source handle and showing its resolved approval path; symbolic links at the file itself are unsupported. Windows holds a source handle denying shared writes/deletion. Unix does not treat FileShare as a guarantee against other writers; Linux/macOS prepare a host-private disk snapshot in the background and verify source content (the snapshot is unlinked right after opening and lives on through the open descriptor, so a host crash midway leaves no plaintext copy in `$TMPDIR`), requiring roughly one extra source-file size of temporary local space. Preparation must not masquerade as uploaded bytes. After transfer, server-side SHA-256 is preferred; pure SFTP accounts without server hashing use a streaming read-back, potentially adding roughly one file size of download traffic. Verification/commit remains running, with unknown speed/ETA rather than a false zero-second completion. The current host cannot reliably validate reconnection identity and namespace, so it explicitly returns `CanResume=false` and rejects resume; there is no automatic checkpoint retry or recovery across application restarts.

Desktop tool cards show a progress bar, transferred/total bytes, percentage, speed, ETA and stage. Unknown totals, preparation and verification use indeterminate progress; failures, cancellation and unknown outcomes are not green successes. Switching chats leaves progress on its original card. Model Stop ends model waiting while the original card can still explicitly cancel. Closing/destroying the panel, plugin deactivation, MCP session deletion/settings reload or Bridge authorization reload explicitly cleans up without claiming remote-stop confirmation. Strict run isolation remains: new model turns cannot query old-run jobs; the original card retains its scope to manage approved uploads.

Bridge pauses the remaining turn budget only during independent upload waiting without other command/monitoring jobs, then resumes the original remaining budget before model requests. That wait also gives up the bridge's global concurrency slot (the chat's own serial gate stays held); before the next model call the turn queues to take back the same gate and resumes the budget only once it has it. Host confirmation has no time limit and a large file can take hours, so holding the slot would leave other chats stuck at the gate. Pure commands/monitoring retain their existing deadlines; manual Stop and lifetime cancellation always apply. When an upload continues after model Stop, Bridge retains a local observer and sends the identified job's final state without restarting the model. Active MCP transfers are not reaped by the two-hour idle policy merely because no tools are called, while completed inactive jobs do not block idle reclamation indefinitely.

AI approval and host upload confirmation are independent layers. Before reading file content or uploading, the host confirms the source plugin, actual target session, local source, remote path and overwrite policy for each job. Only that job is allowed; terminal/session-opening grants do not carry over and there is no Always Allow option. AI `Bypass` cannot skip host confirmation; missing gate, prompt or usable desktop window means rejection. Waiting for confirmation remains part of the independent job lifecycle, not a long RPC. Revoking permissions in plugin management cancels pending and active upload leases, which are rechecked before commit. Revocation withholds new paths/results, but cannot erase previously known inputs/history or promise rollback of a committed file. When the host explicitly denies an upload, desktop/Bridge force the current run to pause with `ExecutionDenied`, without asking the model to find an alternative path. External MCP fails only that call, not its entire fixed client run. If host revocation withholds access to results, pause safely with withheld/`Unknown` rather than repeatedly rereading it as a transient network error. This is a dedicated enforcement point for the new upload API, not a claim that other file capabilities or the universal `remote.files`/local-file Broker are implemented. See [SDK §3.3b](../sdk/sdk-reference.md#33b-independent-upload-jobs-sdk-211) and [IPC §7.2](05-ipc-protocol.md#72-independent-upload-jobs-sdk-211) for protocol and state definitions.

### 1.7 Dual-entry background AI downloads (SDK 2.1.2)

**The download contract is officially published in SDK 2.1.2 on NuGet.org.** Host src / tests central packages, the explicit AI package reference and `minSdkVersion` have all moved to 2.1.2 (`apiLevel: 2`), without local packages or temporary feeds. Downloads are available on Windows and Linux / macOS (x64, arm64) hosts: the commit tests have actually run on Windows and Linux (on ext4 and tmpfs under WSL Debian), macOS only through CI; a real SSH / GUI end-to-end flow is still not claimed as validated.

- **Model entry:** `download_remote_file` removes its former 32 MiB check and starts a background `StartDownloadAsync`. Default `overwrite=false` rejects existing destinations; explicit replacement still requires approval. A given `local_path` follows the same rule as uploads: it must be a fully qualified absolute path (on Windows, `C:foo`, `\tmp\x` and other forms the host would complete from the process's current directory are rejected) and is normalized before approval, so the approval card and host confirmation show the real destination. A structured receipt contains Download direction, state/phase, 64-bit bytes, speed, ETA, generation and stop confirmation, without waiting for the entire file.
- **Remote-link entry:** the former 64 MiB check is removed. After save-path selection, an independent scope and ordinary progress card bind to the conversation/session at click time. This is not a model tool call and fabricates neither model receipts nor terminal notices. The picker does not pre-truncate the final destination or replace per-job host approval. Choosing an existing file has the system save dialog confirm the replacement first; the host is then asked with `overwrite=true`, and its confirmation says the file will be replaced. Chat switches do not mix progress; panel close uses `CloseTransfersAsync`, including cleanup of late starts.
- Both entries share host jobs, bandwidth and resource budgets, not unlimited resources: both directions together allow four actual transfers, 32 active/queued jobs and 1024 lifetime receipts per plugin. An exhausted receipt budget or an operation-identity conflict is a refusal before the host is involved: the tool reports "rejected before execution", which is not an unknown outcome and does not stop the turn with TaskUnknown. Session authorization, model-context/small-file reads, chat attachments and IM platform file limits remain.

`transfer_status` / `transfer_wait` / `transfer_cancel` cover both directions. Short waits default to two seconds, at most 30; expiry or request cancellation ends only that wait, while downloads have no default execution deadline. Explicit cancellation targets the expected generation. Downloads have `CanResume=false`; never automatically redownload Unknown. Cards show upload/download direction, Approval/Transfer/Verification/Commit, bytes, percentage, speed, ETA and cancellation. Unknown totals never invent percentages; full bytes or verification/commit are not success. When the tool got no snapshot (refused, stopped before starting, receipt lost) the card shows a warning instead of a completion check. `Succeeded` requires confirmed final commit. Unknown/TimedOut without stop confirmation does not mean side effects ended; uncooperative backends retain slots/handles until actual exit.

Local observers refresh progress without a model request or context message for every frame. After three consecutive observation failures only that job is reported as "progress cannot be confirmed" (an unknown outcome); the observer then backs off from 1 second, doubling up to 30 seconds, and keeps watching until the host's terminal state, revocation or plugin deactivation, withdrawing the uncertainty as soon as it gets a fresh snapshot. The coordinator delivers each job's terminal notification only once; status/wait consuming it prevents duplicate notices. `transfer_resume` only looks at the original job and does not consume its terminal state, so when it cannot resume, the completion notification still delivers that state to the model. There is never a second tool result for the original CallId. Paths/errors remain untrusted data, and strict owner/run isolation remains. Stop ends model waiting, leaving the job/local observer running and its original card able to cancel explicitly. Panel close/destruction, plugin deactivation, MCP session deletion/settings reload and Bridge authorization reload explicitly clean up. Bridge retains transfer-only remaining-budget suspension, release of the global concurrency slot (see §1.6) and post-Stop local observation/final reporting without automatically restarting the model; command/monitoring budgets are not expanded. External MCP must explicitly query status/wait, with no server-driven model-wakeup promise; active transfers remain protected from idle reclamation.

Download host permission is separate from upload/terminal/session-opening permission, confirming plugin, actual session, remote source, final local destination and overwrite policy for every job. AI Bypass cannot skip it; missing prompt/window means rejection. Revocation cancels pending/active leases, rechecks before commit and redacts subsequent metadata, while cleanup cancellation remains available; committed files are not rolled back. `send_file` refuses registered download destinations before successful commit, rather than sending an old same-name file, and checks both before and after approval: a file whose size or modification time changed during approval is not sent; existing directory restrictions, outbound approval and IM platform limits remain.

**Safe commit supports Windows fixed local NTFS and local filesystems on Linux / macOS (x64, arm64):** the destination directory chain is pinned before approval (Windows holds every ancestor's handle without delete sharing; Linux / macOS walk from `/` with non-following `openat` calls and keep only the parent directory fd), a random temporary file is created exclusively, and the commit is atomic relative to the pinned parent, so renaming an ancestor afterwards cannot redirect it. Default no-replace and approved overwrite never delete the destination first; timestamp preferences target the owned temporary handle. UNC, 9p / SMB / FUSE / NFS and other filesystems fail explicitly before approval, and unsupported backends fail explicitly too; neither falls back to directly writing the final path. Count actual local writes; check length, before/after metadata and available digests, without requiring exec or a second full download for pure SFTP. This is not an atomic remote snapshot guarantee. See [SDK §3.3c](../sdk/sdk-reference.md#33c-independent-download-jobs-sdk-212), [Capability APIs §2.2](07-capability-apis.md#22-independent-download-jobs-sdk-212) and [Security §1.2](12-security-threat-model.md#12-actual-background-download-security-boundaries-sdk-212).

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
