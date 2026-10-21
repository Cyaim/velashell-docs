# 07 · Capability APIs

Everything a plugin can do goes through a capability domain. Each domain maps to an SDK proxy interface on the plugin side, an RPC method family (05), and a host capability service, where the permission `Demand` bridges to an existing subsystem. This document defines the apiLevel 1 domain list and key signatures, in C# from the SDK perspective.

Design discipline:

- Every method documents its required permission. A method without permission is absent from the list and therefore absent from the protocol.
- Everything is asynchronous and cancellable. Long-running operations may include `IProgress<T>`.
- Paths, sessions, and other identifiers use opaque IDs issued by the host. Plugins never access internal objects.
- Bridge implementations **depend only on neutral interfaces in `Core.*`**, such as `Core.Ssh.ISftpClient`, continuing the rule that “library types do not leave Infrastructure”.

## 1. vela.sessions, Sessions

Permissions: `sessions.observe.basic` (redacted list) / `sessions.read` (full) / `sessions.create` (start).

```csharp
public interface ISessions
{
    Task<SessionInfo[]> ListAsync(CancellationToken ct);          // Fields are redacted according to the permission level
    IAsyncEnumerable<SessionEvent> WatchAsync(CancellationToken ct); // Connect/disconnect/reconnect events
    Task<string> ConnectAsync(string savedProfileId, CancellationToken ct); // sessions.create
}
```

## 2. vela.remoteFs, Remote Files (Core Scenarios S1/S2/S7)

Permissions: `remote.files.read` / `remote.files.write`, with scope supporting sessions and path prefixes. The implementation bridges the existing SFTP wrapper. Note the existing semantics: `GetAttributes` returns null for a nonexistent path and does not use an exception to signal existence. The SDK's `StatAsync` likewise returns a nullable value.

```csharp
public interface IRemoteFs
{
    Task<RemoteEntry[]> ListAsync(string sessionId, string path, CancellationToken ct);
    Task<RemoteEntry?> StatAsync(string sessionId, string path, CancellationToken ct);
    Task<Stream> OpenReadAsync(string sessionId, string path, CancellationToken ct);
    Task<Stream> OpenWriteAsync(string sessionId, string path, WriteMode mode, CancellationToken ct);
    Task DownloadAsync(string sessionId, string remotePath, string localPath,   // localPath also requires fs.local.write
                       IProgress<TransferProgress>? progress, CancellationToken ct);
    Task UploadAsync(...);
    Task DeleteAsync/RenameAsync/MkdirAsync(...);
    IAsyncEnumerable<RemoteChange> PollWatchAsync(string sessionId, string path,
                       TimeSpan interval, CancellationToken ct);  // Polling watch, minimum interval 2s (protects the remote)
}
```

Host-side protections: a per-plugin limit on concurrent SFTP operations (4 by default), staggering of bandwidth use against the user's foreground transfers (plugin traffic enters a low-priority queue), and throttled progress callbacks (≥100 ms; the full 100% frame is sent once, only after the transfer succeeds, and never on failure). This incorporates the historical lesson of large-transfer stalls: plugin traffic must not flood the UI scheduler.

### 2.1 Independent upload jobs (SDK 2.1.1)

`IRemoteFsApi` adds `SupportsTransferOperations` and `StartUploadAsync` / `GetTransferAsync` / `WaitTransferAsync` / `CancelTransferAsync` / `ResumeTransferAsync`; see [SDK §3.3b](../sdk/sdk-reference.md#33b-independent-upload-jobs-sdk-211) for the complete contract. Unlike synchronous transfers above, start returns a snapshot while the host continues streaming content. A query/wait deadline does not cancel the transfer, and the legacy two-hour RPC limit is not an execution deadline.

The current host permits four concurrent uploads and 32 active/queued jobs per plugin, retaining up to 1024 lifetime idempotency receipts. Quota exhaustion rejects new jobs; reacquiring an existing id is not blocked by the active-job quota. These are resource/concurrency budgets, not file-size limits.

The new upload API uses dedicated trusted host confirmation and a per-job authorization lease. It displays the plugin, actual session, local source, remote destination and overwrite policy, with only Allow This Job or Deny; terminal/session-opening grants never carry over. Missing gate, prompt or desktop window means rejection, and waiting for confirmation does not block the start RPC. Plugin permission revocation cancels pending/active uploads, prevents subsequent commit and withholds new job metadata. An uncertain result after commit has begun still means `Unknown`, not a rollback promise. AI `Bypass` skips only plugin approval, never this host confirmation.

Progress uses 64-bit byte counters, monotonic snapshot sequence, execution generations and explicit terminal states. Full bytes can still mean remote finalization; only `Succeeded` means success. Cancellation targets a generation; resume requires fresh approval and validation of connection identity, source object, checkpoint and final commit outcome. Never automatically replay an uncertain side effect. New methods explicitly default to unsupported: capability false must not fall back to synchronous large-file upload. This contract does not implement the universal permission Broker; interpret actual authorization and lifecycle boundaries according to the host implementation.

## 3. vela.remoteExec, Remote Execution (S5/S6)

Uses a **separate execution channel**, never the user's terminal or its shell history/environment. The actual interface is `IRemoteExecApi`; see the [SDK reference §3.3a](../sdk/sdk-reference.md).

| Method | Actual contract and limits |
| --- | --- |
| `RunAsync` | Returns `ExecResult.Output` / `Error` / `ExitCode`; nonzero exit is a result, not an exception. Default 30 seconds, maximum 10 minutes |
| `StreamAsync` | stdout/stderr callbacks through `IProgress<ExecOutput>`, returning `ExecStreamResult`; no execution deadline by default, caller cancellation or explicit Timeout ends execution; no stdin |
| `StreamSudoAsync` (SDK **2.1.0**) | Returns `SudoExecResult`; `IProgress<SudoExecEvent>` carries non-secret states and merged raw PTY text fragments. Linux sudo only; the entire command runs as root, including pipes/redirections. Not a general interaction API |

Ordinary and sudo streams share the limit of four per plugin. Receivers must forward synchronously and quickly, not reorder output with `Progress<T>`. A slow receiver does not abort execution; it slows the remote stream instead: when an isolated plugin's notification queue is full the host waits for space, and backpressure travels back to the remote through the SSH window, while in-process plugins are already called synchronously on the line-reading thread. Ordinary output without newlines is split into fragments of at most 4096 characters; CR may delimit fragments, and long lines carry `[VelaShell: continued]`. Do not assume one callback is one complete application record. Like ordinary streams, sudo output replaces non-UTF-8 bytes with U+FFFD, so GBK logs, binary content or non-UTF-8 file names do not turn a running root command into Unknown.

**Separate current permission behavior from design goals.** This blueprint's `remote.exec` Broker, universal Demand and non-disableable audit are design goals, not guarantees delivered here. The current `PluginPermissionGate` covers `terminal.write` / `session.open` and dedicated per-job leases for the new upload API (§2.1), not generic grants for other remote file operations. This change adds **trusted host confirmation on every sudo call**, not a complete generic permission system. Source identity is bound from the host manifest and the real target comes from the connection; even NOPASSWD cannot skip confirmation.

AI-plugin approval and host confirmation are separate layers. `Ask` / `ReadOnlyAuto` use the distinct `run_command_elevated` kind, display the target and whole-command root scope, and have no RepeatKey. Missing approval Handler means rejection; even root `cat` is not read-only auto-approved. Explicit user-selected `Bypass` skips only plugin approval, never host confirmation. MCP without a Handler rejects in the first two modes; Bypass may request confirmation from the desktop host.

Only the host's trusted masked dialog collects the password: no login-password reuse, remembering, SDK/RPC/plugin/model/history/log exposure. The controlled PTY must confirm echo is off and match this call's random prompt before sending it. Once the command starts, password responses are permanently disabled and stdin is closed. The complete raw startup marker must not appear directly in argv that sudo may echo; only an authorized root wrapper may generate it. A sudoers refusal diagnostic must not be mistaken for Running/Exited or leak authentication-tail content as command output. Complex PAM/OTP, missing UI, rejected PTY or sudoers refusal of the root-shell wrapper fails explicitly; no policy bypass or ordinary-execution fallback. A fish or tcsh login shell works too: the login shell receives only a `/bin/sh` invocation with the controlled script encoded in a `printf %b` payload, which it cannot rewrite (see [Security §1.1](12-security-threat-model.md#11-actual-boundaries-of-this-sudo-and-managed-ai-execution-change-sdk-210)). Ordinary commands containing sudo are not regex-rewritten or automatically rerun elevated, avoiding replay of an already-executed script prefix. This is not terminal autofill and does not change existing terminal credential decisions.

See [SSH specification §9.1](../ssh/spec/08-failures.md#91-connection-wide-payload-suppression-leases-for-sensitive-interactions) for diagnostic leases and conservative asynchronous teardown, and [Security §1.1](12-security-threat-model.md#11-actual-boundaries-of-this-sudo-and-managed-ai-execution-change-sdk-210) for scope-cache/revocation limits and unverified areas.

## 4. vela.terminal, Terminal (S3/S4)

Permissions: `terminal.read` / `terminal.write` (high sensitivity).

```csharp
public interface ITerminal
{
    IAsyncEnumerable<TerminalOutput> SubscribeAsync(string sessionId, CancellationToken ct);
        // Host-side frame-coalescing throttle ≥16 ms; call Demand once when the subscription is established, and cut the stream on revocation
    Task<TerminalSnapshot> GetSnapshotAsync(string sessionId, CancellationToken ct);  // Screen + tail of scrollback (line limit)
    Task<string?> GetSelectionAsync(string sessionId, CancellationToken ct);
    Task WriteAsync(string sessionId, string input, CancellationToken ct);
        // Through the existing input serialization queue (do not write directly to the stream); ≤4 KB per call by default
    Task<string> CreateOutputChannelAsync(string title, CancellationToken ct);
    Task AppendToChannelAsync(string channelId, string text, CancellationToken ct);
        // Plugin-specific read-only output page (attached as a VelaDock document); requires no terminal permission, only ui.contributions
}
```

Additional `terminal.write` safeguards: injected input is briefly marked in the terminal with a superscript identifying the source plugin, which the user can disable in settings. Per-plugin write frequency is rate-limited (20 calls/min by default to prevent runaway screen spam); exceeding the limit trips a circuit breaker and notifies the user.

## 5. vela.localFs, Local Files

Permissions: `fs.local.read` / `fs.local.write` (scope = directory prefix). **Permission-free paths**: paths returned by `PickFileAsync` / `PickFolderAsync`, where the host displays the system picker, automatically receive temporary authorization. The user made the selection directly, following the Android SAF model. Plugins are encouraged to prefer the picker over requesting a broad directory scope.

```csharp
public interface ILocalFs
{
    Task<string?> PickFileAsync(FilePickerOptions opts, CancellationToken ct);
    Task<string?> PickFolderAsync(..., CancellationToken ct);
    Task<Stream> OpenReadAsync(string path, CancellationToken ct);
    Task<Stream> OpenWriteAsync(string path, WriteMode mode, CancellationToken ct);
    Task<LocalEntry[]> ListAsync(string path, CancellationToken ct);
    // Regular Stat/Delete/Move/Copy…
}
```

## 6. vela.storage / vela.secrets / vela.settings

```csharp
public interface IStorage      // Permission: storage.private (normal)
{
    Task<T?> GetAsync<T>(string key);   Task SetAsync<T>(string key, T value);
    Task RemoveAsync(string key);       string DataDirectory { get; }   // Direct file access is also restricted to this directory
}
public interface ISecrets      // Permission: secrets (dangerous); namespace forced to <pluginId>/
{
    Task<string?> GetAsync(string name); Task SetAsync(string name, string value); Task DeleteAsync(string name);
}   // Uses the OS credential store underneath (DPAPI/Keychain/libsecret), a separate namespace from host credentials and invisible to each other
public interface IPluginSettings   // Permission: settings.own (normal)
{
    T Get<T>(string key);  Task SetAsync<T>(string key, T value);
    IAsyncEnumerable<SettingChange> WatchAsync(CancellationToken ct);   // Pushed when the user changes a setting in the settings page
}
```

## 7. vela.ui, UI (See 08)

Permission: `ui.contributions` (normal). Includes commands/menu items/status-bar dynamic updates, VelaUI surfaces (panels and document pages), dialogs (Message/Confirm/Input/QuickPick), in-app notifications (rate limit: ≤6 per plugin per minute), and progress indicators (in the status bar or notifications).

## 8. vela.events, Host Event Subscriptions (S5)

Permission: most event payloads follow the permission for the corresponding domain, such as session events requiring `sessions.read`. `appStartup/appShutdown/themeChanged/localeChanged` require no permission.

```csharp
public interface IEvents
{
    IAsyncEnumerable<HostEvent> SubscribeAsync(EventFilter filter, CancellationToken ct);
    // HostEvent: SessionConnected/Disconnected, TransferCompleted, ThemeChanged,
    //            LocaleChanged, AppShutdown (gives the plugin a chance to flush), ScheduleFired (see 11)
}
```

## 9. vela.audio, Audio Output (S2)

Permission: `audio.playback`. Design choice: **the plugin decodes, the host outputs**. The host does not include codecs, avoiding codec dependency bloat and licensing issues. It only provides PCM mixing output. MP3 decoding is supplied by the plugin using a managed decoder, such as NLayer, as demonstrated by the official sample.

```csharp
public interface IAudio
{
    Task<IAudioTrack> CreateTrackAsync(AudioFormat fmt, CancellationToken ct); // fmt: sample rate/bit depth/channels
    // IAudioTrack: WriteAsync (PCM chunks, backpressure), pause/resume/stop, volume, position feedback
}
```

Host output backend: pending a selection spike, with candidates including a miniaudio binding, OpenAL Soft, and three platform-native implementations using WASAPI + CoreAudio + ALSA. The placeholder interface comes first, so selection does not block other domains. Global mixing applies at a given time. Plugin volume is constrained by the host master volume and mute switch. When a plugin is disabled or crashes, its audio track is muted and reclaimed immediately.

## 10. vela.net, Network (Declarative)

Permission: `network` + a domain-list declaration in the manifest. v1 provides no physical interception (see the premise in 06). The SDK provides `ctx.Http`, a convenience wrapper based on HttpClient that automatically includes the plugin UA. Requests made through the SDK wrapper are recorded through the host audit channel with the target domain. Directly constructing an HttpClient cannot be prohibited in v1. Compliance depends on review and signatures, and the documentation states this limitation plainly.

## 11. vela.clipboard / Notifications / i18n

- `clipboard.read/write`: read/write the system clipboard. Read is a dangerous permission because the clipboard often contains passwords.
- `notifications.system`: OS-level notifications. In-app notifications belong to vela.ui and do not require this permission.
- `i18n.read`: the current locale and translated common host strings, such as the “OK” and “Cancel” buttons, so plugin copy matches the host.

## 12. vela.ai, AI Gateway (Longer Term, See 11-automation-and-ai)

Permission: `ai.invoke`. Reserved in apiLevel 1, with the interface held for future use and the host able to declare the capability unavailable. Full implementation is covered in [11-automation-and-ai.md](11-automation-and-ai.md).

## 13. Development Plan (This Work Item)

Batch divisions correspond to the milestones in 14:

| Task | Content | Dependency | Estimate |
| --- | --- | --- | --- |
| C-1 | Capability-service foundation: registry, Demand base class, audit hooks, per-plugin rate limiter | B-2, P-3 | 3d |
| C-2 | First batch: storage / settings / secrets / events (foundation domains, minimum dependencies for sample plugins) | C-1 | 4d |
| C-3 | Second batch: sessions / remoteFs (bridge Core.Ssh, low-priority transfer queue, progress throttling) | C-1 | 5d |
| C-4 | Third batch: localFs (picker temporary-authorization model) + clipboard + notifications | C-1 | 3d |
| C-5 | Fourth batch: terminal (subscription frame coalescing, snapshots, write-rate circuit breaker) + remoteExec (auditing) | C-3 | 5d |
| C-6 | Audio-output spike (evaluation report for three candidates) → IAudio implementation | C-1 | 5d |
| C-7 | vela.net SDK wrapper + auditing | C-1 | 1d |
| C-8 | Contract test suite: for every domain, “permission matrix × normal/boundary/cancellation/disconnection” cases (connects to 13) | Each batch | Ongoing |

Acceptance: the domains required by S1 (C-2/C-3) support the image-viewer sample end to end. Once C-5 is complete, all domains for S3/S4/S5 are available.
