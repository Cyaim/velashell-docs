# Credential manager integration: design and implementation plan

> Date written: 2026-09-15 (v2 revised the same day; v3 2026-10-03)　　Baseline code: `dev` @ `4ee4c982`　　Status: **v3: the first source, the built-in "shared credentials", has shipped (#550, see §0.1); the three external password managers (phases 1a / 1b / 1c) and the system keychain (phase 2) are still to be implemented**
>
> Purpose: stop connection profiles from storing the password itself; instead they store a **reference** to an item in 1Password / Bitwarden / KeePassXC /
> the system keychain, and the password is fetched only at the moment of connecting.
>
> **Verification scope** (v2):
> - Code: the connection path, settings and UI, and wiring and tests were combed through file by file, and the key conclusions were spot-checked against the source.
>   Line numbers are based on `4ee4c982` and drift with later commits; **symbol names are authoritative**.
> - CLIs: checked against the three vendors' official documentation and open-source code (links at the end of each part of §8).
>   **Not yet tested on real machines** (`op` / `bw` / `keepassxc-cli` were not installed where this was written); items marked "to be tested" must be run before phase 1 starts.
>
> Related documents: [System-keychain credential storage + sudo prompt autofill: feasibility research](system-keychain-and-sudo-credential-fill-feasibility-research.md) (the "other entry point" to this document),
> [Ecosystem ideas §2.4 credential providers](../plugins/15-ecosystem-ideas.md), [Security threat model](../plugins/12-security-threat-model.md).

## Contents

- [0. Conclusion first](#0-conclusion-first)
  - [0.1 v3: built-in shared credentials have shipped (#550)](#01-v3-built-in-shared-credentials-have-shipped-550)
- [1. Correcting a premise](#1-correcting-a-premise)
- [2. Current state (verified)](#2-current-state-verified)
- [3. Existing defects found along the way](#3-existing-defects-found-along-the-way)
- [4. Decisions made](#4-decisions-made)
- [5. Architecture](#5-architecture)
- [6. Unlock session cache](#6-unlock-session-cache)
- [7. CLI runner](#7-cli-runner)
- [8. Specifications of the three providers](#8-specifications-of-the-three-providers)
- [9. System keychain provider](#9-system-keychain-provider)
- [10. Settings and UI](#10-settings-and-ui)
- [11. Out of scope](#11-out-of-scope)
- [12. Phasing and task list](#12-phasing-and-task-list)
- [13. Test plan](#13-test-plan)
- [14. Documentation and repository sync checklist](#14-documentation-and-repository-sync-checklist)
- [15. Risk register](#15-risk-register)
- [16. New decisions awaiting confirmation](#16-new-decisions-awaiting-confirmation)
- [17. Small questions to settle during implementation](#17-small-questions-to-settle-during-implementation)
- [18. Revision history](#18-revision-history)

---

## 0. Conclusion first

**Decisions made (2026-09-15)**

| Item | Final |
|---|---|
| Abstraction | Add `ICredentialProvider` (the credential **source**); do **not** rework `ISecretProtector` (credential **encryption**) |
| First providers | 1Password (`op`), Bitwarden (`bw`), KeePassXC (`keepassxc-cli`) — **all three delivered together** |
| Private keys | **Go through the SSH agent**; private keys are not fetched through the CLIs. Implemented separately once the upstream PR is merged |
| Reference granularity | **A whole profile binds to one item**; an advanced option lets this profile's username override the item's username<br>**v3:** there is no longer an "override username" switch — if the connection fills in a username, that one wins; if it is left empty, the item's is used (§0.1) |
| Unlock secret | **In-memory TTL cache**, with a configurable duration |
| Resolution failure | **Fall back to the login dialog + a notice bar at the top** |
| Reference and stored password | **Never coexist** |
| Shape | **Built into the host**; no plugin SDK contract is opened<br>**v3:** the first source is the host's built-in "shared credentials" (provider id `shared`); the three external CLIs will later plug into the same resolver as further sources |
| Relation to `IMasterKeyStore` | **Proceed in parallel**; the system keychain provider waits for its P/Invoke layer |

**Key conclusions added in v2**

1. **Plaintext passwords have three carriers and five construction points**, not just `BuildChainAsync` (§2.1). FTP and the three kinds of plugin protocol do not go through `ConnectionWorkflowService`, so the resolver must be wired into every construction point.
2. **Four existing defects were found** (§3); one of them means that even with "Remember password" unticked, a password the user typed by hand still gets written to the database. It should be fixed separately as **phase 0a**; otherwise this feature's "profiles with a reference never persist a password" would be built on a leaking foundation.
3. **Automatic reconnect must not show an unlock dialog.** After the network comes back, N tabs reconnect at once and would trigger unlocking at once. Resolution has to distinguish interactive from non-interactive (§5.7).
4. **The invariant "profiles with a reference never persist a password" is enforced in the repository layer.** The repository is the single exit for every save, which is more reliable than guarding each call site one by one (§5.4).
5. **On Windows, 1Password authorizes per process chain, and this is not clearly documented.** Whether VelaShell launching `op` shows Windows Hello every time is the **first item to test** in phase 1, and the result directly decides how many calls the design can afford (§8.1).
6. **Bitwarden's `bw list items` outputs the passwords along with everything else**; **the KeePassXC CLI can only address entries by path and takes the first match when titles collide** (§8.2 / §8.3).

### 0.1 v3: built-in shared credentials have shipped (#550)

> 2026-10-03. Origin: host issue #550 "several connections sharing one account / key (see WindTerm's OneKey)" —
> changing the password of a hundred devices meant changing it a hundred times. It has the same skeleton as this document: the profile stores a **reference**, and the resolver fetches it at the moment of connecting.
> So this document's foundations (phases 0a / 0b) were done first, with the host's own "shared credentials" as the **first source**; the three external CLIs will plug in later as further sources.
> The host-side record is in `plan.md` in the code repository; the UI specs are in the settings window, connection profile and login dialog sections of [Interaction and UI specs](interaction-and-ui-specs.md).

**What has shipped**

| Layer | Contents | Location (host repository) |
|---|---|---|
| Core | `CredentialReference` (`ProviderId` + `ItemId`), `ICredentialProvider`, `ICredentialResolver`, `ResolvedCredential`, `CredentialProviderException` / `CredentialFailure`, `CredentialMaterial` (deciding and clearing what was typed by hand), the `SharedCredential` model, `SessionProfile.CredentialSource` | `src/VelaShell.Core/Credentials/`, `Core/Data/ISharedCredentialRepository.cs` |
| Infrastructure | `SonnetDbSharedCredentialRepository` (collection `shared_credentials`; passwords and private-key passphrases encrypted through `ISecretProtector`, as strongly as connection passwords), `SharedCredentialProvider` (provider id `shared`), `CredentialResolver`; DI registers the source collection by "register the concrete type first, then forward" | `src/VelaShell.Infrastructure/Credentials/`, `Persistence/` |
| Presentation | `ConnectionWorkflowService` resolves every hop; `SharedCredentialService` (who uses it, attach / detach, saving the editor, delete) | `src/VelaShell.Presentation/Services/` |
| App | Settings → Shared Credentials page, the credential editor, "Credential source" on the connection profile page, the login dialog's notice bar and "Also update the shared credential" | `src/VelaShell/` |

**Differences from the v2 design** (the sections below keep the v2 text as the basis for external sources; where this section is authoritative, each point is listed here)

| v2 design | v3 reality | Reason |
|---|---|---|
| Decision 2: private keys go through the agent; a provider supplies only username and password | `ResolvedCredential` also carries `DisplayName`, `AuthMethod`, private-key path, passphrase and certificate path. **External vaults still fill in only username and password**; decision 2 still holds for them | What shared credentials need to share is very often "the same private key + passphrase"; storing a private-key path in the local store does not carry the concerns of the external-CLI layer |
| Decision 3: an `OverrideUsername` switch (§5.3) | No switch: **if the connection fills in a username, it overrides; if left empty, the credential's is used**; a credential may also carry no username, leaving it to each connection | One switch fewer, while covering "one set of keys with different login names" (WindTerm users asked for the same thing on OneKey) |
| §5.3 the reference contains `Scope` / `DisplayName` / `DisplayUsername` / `OverrideUsername` | Only `ProviderId` + `ItemId` | The local source has no use for the other fields; they will be added when external sources arrive — JSON omits nulls when writing, so adding fields needs **zero migration** |
| §5.4 the invariant strips only `Password` | All authentication material is stripped: `Password`, `PrivateKeyPath`, `PrivateKeyPassphrase`, `CertificatePath` (username and authentication method untouched; the plugin secrets dictionary is still saved encrypted as usual) | The rule "material in memory = typed by hand this time" (`CredentialMaterial.HasInline`) has to hold for all four authentication methods |
| §5.2 `ResolveAsync` returns `ResolvedCredential?` and the caller fills it in | `ICredentialResolver.ResolveAsync(SessionProfile, ct)` directly returns **a copy of the profile for this one connection**; with no reference, or with material typed by hand plus its own username, it is returned unchanged; with material typed by hand but the username following the credential, only the username is filled in from the credential | Filling the fields at five construction points is five places to forget something; the copy is never written back, so plaintext never reaches the session-tree cache or the repository |
| §5.5 the resolver is injected into the Infrastructure services at the three plugin construction points | FTP and the three kinds of plugin protocol (file / terminal / workspace, including the "Test" probe) are resolved in `MainWindowViewModel` before the service is called | All callers of those three services are there; one helper method covers them all, instead of giving each of the three services another dependency |
| §5.5 a jump-host resolution failure "ends as a connection error" | Implemented as: on the jump-host hops, `CredentialProviderException` is rethrown as `InvalidOperationException` ("Jump host X: reason"), **with no fallback to the login dialog**; only when the target host's own credential cannot be fetched does it fall back to the login dialog | The login dialog asks for the target host's credentials; using it to stand in for a jump host would just fail once more |
| §5.6 per-attempt cache, unlocking serialized per provider; §5.7 interactive / non-interactive | **Not implemented**: the local source has no external process, no unlocking and no timeout | Done together with external sources (phase 1a); `CredentialResolveOptions` will be added to `ResolveAsync` then |
| §2.3 decision points: "a reference counts as not missing a password" | Profiles with a reference **never prompt up front** (`profile.CredentialSource is not null`); whatever is missing is reported at resolution time, then it falls back to the login dialog | A reference profile with material typed by hand should not prompt up front either, so the criterion was collapsed into one |
| §5.8 notice bar in the login dialog | Done: three explanations — rejected / incomplete / no longer exists; an "Also update the shared credential (used by N connections)" checkbox — ticked by default when incomplete, unticked when rejected (it may be just this one host with a different password), not offered when the credential no longer exists; "Remember password" hidden for reference profiles | "The moment of rejection" is where this feature really saves effort: after changing the server password, type it once, tick the box, and every connection referencing it switches over |
| §10.2 a new "Credential managers" settings page | The new settings page is called "Shared Credentials" (right after "Key Management"); configuration of external sources will go on the same page later | For now the page only has local credentials |
| Gist sync "no, no code change needed" (§5.10) | The payload gains `sharedCredentials` (passwords and passphrases stripped when unencrypted, the locally stored ones kept on pull; credentials are written before connections); **when shared credentials are used**, the payload and envelope versions are written as 3, otherwise still 2; this device supports up to 3 | An older client that does not understand references would drop them as unknown fields, and pushing back to the cloud would then erase the references; writing 3 makes older clients refuse it by the rules, while people not using this feature still interoperate with older versions |

**Still not done**: phases 1a / 1b / 1c (CLI runner, unlock cache, the three providers, item picker, unlock dialog) and phase 2 (system keychain provider); D2 in §3 (private-key passphrases are not governed by "Remember password", pending §16 N2) and D4 (`SshSession.ConnectionInfo` holds secrets for a long time; plaintext resolved from shared credentials also lingers there).

---

## 1. Correcting a premise

`feature-plan.md` originally said: "`ISecretProtector` is that abstraction layer; just add an `ICredentialProvider` to make it pluggable". That is only half right.

- `ISecretProtector` is about **how to encrypt**: a synchronous `string → string` transformation.
- Credential manager integration is about **where the secret lives and who owns it**: the profile stores a reference, not the value.

These are different dimensions and cannot share one interface:

| If `ISecretProtector` called `op` / `bw` | Consequence |
|---|---|
| Every load in `GetAllSessionsAsync()` calls `Unprotect` on **all** profiles (`SonnetDbSessionRepository.cs:38`) | Starting up or refreshing the session tree would spawn N CLI processes, possibly with N biometric prompts in a row |
| The interface is synchronous, and there are 5 callers (sessions, Gist PAT, two plugin-secret sites, plugin trust) | Either async spreads through the whole codebase, or the UI thread blocks waiting for an external process |
| `PluginTrustRepository` uses the AES-GCM authentication tag as its integrity boundary (`PluginTrustRepository.cs:37`) | With an external vault, that boundary no longer exists |
| When `Unprotect` cannot decrypt, it **returns the input unchanged** (`AesSecretProtector.cs:79`) | On a resolution failure, `op://vault/item` would be sent to the remote server as the password |

The correct structure is two seams, each owning one layer:

```
                ┌──────────── Seam B: credential source (this document) ─────────┐
 SessionProfile │ stores a "reference" CredentialReference {provider, item}      │
                │        │ resolved only at the moment of connecting             │
                │        ▼                                                       │
                │ ICredentialProvider ─ 1Password / Bitwarden /                  │
                │                       KeePassXC / system keychain              │
                └────────────────────────────────────────────────────────────────┘
                ┌──────────── Seam A: master-key storage (keychain research 0/1) ┐
 Stored         │ ISecretProtector (unchanged) ← IMasterKeyStore                 │
 passwords      │                                ├ DPAPI / Keychain /            │
                │                                └ libsecret / file fallback     │
                └────────────────────────────────────────────────────────────────┘
```

What "two entry points to the same thing as the system-keychain research" means concretely:

- The **`IMasterKeyStore`** of the research's phases 0/1 protects the ciphertext **that VelaShell itself stores**; it and this document do not depend on each other and proceed in parallel.
- **The system keychain as an `ICredentialProvider`** lets users reference items they **already have** in the system credential manager. It shares the same P/Invoke layer as `IMasterKeyStore`, so it comes later.
- The credential picker of the research's **phase two (sudo fill)** reuses the providers' `SearchAsync` / `ResolveAsync` directly.

---

## 2. Current state (verified)

### 2.1 The three carriers of plaintext passwords

| Carrier | Construction point | Goes through `ConnectionWorkflowService`? | Delivered to |
|---|---|---|---|
| `ConnectionInfo` (`Core/Models/ConnectionInfo.cs`) | `ConnectionWorkflowService.BuildChainAsync` (`:213`; direct `:224`, chained `:249`). Each jump hop is re-read from the repository (`:241`) | Yes | `InfrastructureServiceCollectionExtensions.AddCredential` (`:327` → `new PasswordCredential(ci.Password ?? "")`); jump hosts through `BuildProxyChain` (`:445`) |
| `FtpConnectionInfo` (`Core/Ftp/FtpConnectionInfo.cs`) | `FtpConnectionInfo.FromProfile` (`:50`); the only production call site is `MainWindowViewModel.cs:3684` | **No** | `FtpFileService.CreateConnectedClientAsync` (`:627`; proxy route `:639`) |
| SDK request DTOs | `PluginProtocolFileService.OpenSessionCoreAsync` (`:100-108`, `ProtocolConnectRequest`)<br>`PluginTerminalShellStream` (`:218-226`, `ProtocolConnectRequest`)<br>`PluginWorkspaceLauncher.OpenCoreAsync` (`:127-140`, `WorkspaceConnectRequest`) | **No** | Code of **in-process plugins**. An isolated-mode PluginHost may not register protocols or workspaces (`RemotePluginContext.cs:92-102,155-181`) |

`ConnectionInfo` is built only in `BuildChainAsync` (within `src`). So the SSH side has just this one point, FTP and the plugin protocols have four more, **five construction points in total**.

### 2.2 All connection entry points

The "Resolution point" column says where this design turns the reference into a password (see §5.5).

| Entry point | Location | Profile instance used | Resolution point |
|---|---|---|---|
| SSH terminal | `TryConnectProfileAsync` `MainWindowViewModel.cs:3194-3342` → `RunHandshakeAsync` `:2304` → `ConnectProfileAsync` `:2316` | Passed in by the caller | `BuildChainAsync` |
| In-place reconnect (Ctrl+R / Enter / the bubble's button / automatic reconnect / resume after sleep) | `ReconnectTabAsync` `:2394-2518` → `ConnectProfileAsync(tab.Profile)` `:2455`; callers: command `:872`, `ReconnectRequested` `:1246/2259`, bubble `:2782`, the `OnTabDisconnected` timer `:2812-2832`, `ReconnectAllAfterResume` `:2694-2713` | `tab.Profile` (in memory) | `BuildChainAsync`; **automatic reconnect must use non-interactive mode** (§5.7) |
| Clone tab | `:879-895` → `TryConnectProfileAsync(ActiveTerminalTab.Profile)` `:888` (the same instance, not `Clone`d) | `tab.Profile` | Same as above |
| Standalone SFTP document | `OpenSftpDocumentForProfileAsync` `:3476-3571` → `ConnectProfileAsync` `:3519` | `current` in the loop | `BuildChainAsync` |
| Tunnel panel background connection | `ConnectTunnelHostAsync` `:1767-1791`; tunnel automatic reconnect `TunnelPanelViewModel.AutoReconnectAsync` `:591-600` | The tunnel panel's server list | `BuildChainAsync`; automatic reconnect uses non-interactive mode |
| SSH jump tunnel of workspace plugins | `EstablishWorkspaceTunnelAsync` `:4276-4339` (the jump profile is read from the repository `:4304`; an already connected session may be reused `:4310`) | Repository instance | `BuildChainAsync` |
| Recent connections / command palette / session tree / startup restore / "Connect" in the profile dialog | `TryConnectRecentAsync` `:4531`, palette `:2192`, `SessionTreeViewModel` → `MainWindow.axaml.cs:446`, restore `MainWindow.axaml.cs:526-578`, dialog `MainWindow.axaml.cs:1187` | Each one's own cache / repository instance | Routed by protocol |
| External launch (URL / command line / `.xsh`) | `ExternalLaunchCoordinator.cs:66` → `ResolveProfileAsync` `:223-268` (when the request carries no password, the saved profile is reused `:225`) | Saved or temporary profile | Routed by protocol |
| Plugin `Sessions.OpenAsync` (the AI plugin's `open_session`) | `SessionsCapability.OpenAsync` `:91-145` → `HostSessionOpener.cs:48` → `TryConnectProfileAsync` | Repository instance; the plugin only gets a `SessionInfo`, **with no credentials** | `BuildChainAsync` |
| Test connection | `ConnectionProfileViewModel.cs:1202` → `TestConnectionAsync` `ConnectionWorkflowService.cs:57-88` | A new object from `BuildProfile()` | `BuildChainAsync`; plugin protocols go through `ProbePluginConnectionAsync` `:4201-4245` |
| Connection diagnostics step 4 | `ConnectionDiagnosticsService.cs:236` → `TestConnectionAsync`; `HasStoredCredentials` `:351-360` decides whether to skip | Tree cache or `tab.Profile` | `BuildChainAsync` |
| FTP / FTPS | `OpenFtpDocumentForProfileAsync` `:3641-3739` → `FromProfile` `:3684` | `current` in the loop | **In `MainWindowViewModel`, before `FtpConnectionInfo` is built** |
| Plugin file-system protocols (S3 and others) | `OpenPluginDocumentForProfileAsync` `:3771-3903` → `_pluginProtocols.OpenSessionAsync` `:3834` | `current` in the loop | `PluginProtocolFileService` |
| Plugin terminal protocols (Telnet / serial) and their reconnect | `OpenPluginTerminalForProfileAsync` `:3923-3960`; reconnect `ReconnectPluginTerminalAsync` `:3997-4033` | `tab.Profile` | `PluginTerminalShellStream` |
| Plugin workspaces (Redis and others) | `OpenWorkspaceDocumentForProfileAsync` `:4055-4185` → `launcher.OpenAsync` `:4124` | `current` in the loop | `PluginWorkspaceLauncher` |

**Panels that reuse an existing connection without re-authenticating** (unaffected): the SFTP sidebar and remote editing (channels opened on the existing client, `InfrastructureServiceCollectionExtensions.cs:181-195`), resource monitoring, process management, tunnels, route tracing.

`SessionProfile.AutoStartTunnelIds` currently **has no reader at all**; there is no "auto-start tunnels" connection path.

### 2.3 Where "missing credentials" is decided

| Decision | Location | Callers | Change in this design |
|---|---|---|---|
| `RequiresCredentials` | `MainWindowViewModel.cs:3174-3187` | `:1777` tunnels, `:3245` SSH terminal, `:3496` SFTP | A reference counts as not missing a password; with `OverrideUsername == false` it also counts as not missing a username |
| `RequiresPluginCredentials` | `:4433-4436` | `:3812`, `:4095` | Same as above |
| `RequiresFtpCredentials` | `:4492-4494` | `:3661` | Same as above |
| `ConnectionWorkflowService.ValidateProfile` | `ConnectionWorkflowService.cs:258-307`; **throws** when the password is empty (`:286-289`) | `ConnectProfileAsync` `:94` (credentials required), `SaveProfileAsync` `:40` (not required) | Skip the non-empty password check when there is a reference |
| `ConnectionDiagnosticsService.HasStoredCredentials` | `:351-360` | Step 4 `:223-231` | A reference counts as having credentials |
| `ExternalLaunchCoordinator` | `:225` | — | No change needed: when the request carries no password the saved profile is reused, and it is enough for the saved profile to carry a reference |

**Paths with no decision and no prompt**: `ReconnectTabAsync`, opening and reconnecting plugin terminals, `ProbePluginConnectionAsync`, the jump connection of `EstablishWorkspaceTunnelAsync`, and each jump hop in `BuildChainAsync`.

### 2.4 Retry loops and the login dialog

- **Dialog delegate**: `MainWindowViewModel.InteractiveAuthenticator` (`:589`) is set to `PromptCredentialsAsync` in `MainWindow.axaml.cs:407`. The latter is serialized through the `_connectPromptGate` semaphore (`:1479-1491`), and its core is `PromptCredentialsCoreAsync` (`:1496-1546`).
- **⚠️ The dialog modifies the instance passed in, in place, and returns the same instance** (`:1526-1545`): `Username`, `AuthMethod`, `Password` (`:1533`), `RememberPassword` (`:1535`), `PrivateKeyPath`, `PrivateKeyPassphrase`, `CertificatePath`. The instance passed in may be the session-tree cache, the command-palette cache, `tab.Profile` or an item in the tunnel panel's list.
- **Five retry loops**, all a local `for (int attempt = 0; attempt < 3; attempt++)`:

| Loop | Location | Handling of authentication failure | Certificate trust |
|---|---|---|---|
| SSH terminal | `:3243-3341` | `VelaSshAuthenticationException` with a dialog delegate present → `continue` (`:3324-3328`) | — |
| SFTP | `:3494-3570` | Silent disconnect, record it, then `continue` (`:3553-3558`) | — |
| FTP | `:3659-3738` | `VelaFtpAuthenticationException` → `continue` (`:3723`) | After trusting, `PersistProfileIfSavedAsync` + `attempt--` (`:3712-3714`) |
| Plugin file system | `:3810-3902` | `PluginProtocolAuthenticationException` → `continue` (`:3887`) | Same as above, at most 2 extra attempts (`:3872-3878`) |
| Plugin workspace | `:4093-4184` | Same as above (`:4169`) | Same as above (`:4152-4161`) |

- The decision is always `needsPrompt = attempt > 0 || Requires*(current)`, so after the first failure a prompt is certain.
- The "Retry" closures on the failure cards of the four document paths all capture **the `profile` originally passed in** (`:3492,3657,3787,4069`). The only reason a retry today does not need the password typed again is that the dialog modified that instance in place.

### 2.5 All persistence points

The repository has a single implementation, `SonnetDbSessionRepository`: `SaveSessionAsync` → `Protect` (`:131-140`) clones first and then encrypts; reads `Unprotect` (`:167-173`) **decrypt in place**, so whatever any repository read returns is plaintext. When `SaveSessionAsync` updates an existing profile, it keeps the directory bookmarks already in the store rather than taking the ones passed in (the callers in the table below that write back a whole profile mostly hold stale copies); Gist sync pull and backup import, which must overwrite bookmarks too, go through `ReplaceSessionAsync`, and changing only bookmarks goes through `UpdateDirectoryBookmarksAsync` (other fields untouched).

| Save point | Location | Honours "Remember password"? |
|---|---|---|
| `ConnectionWorkflowService.SaveProfileAsync` | `:41` | Yes (through `WithPersistablePasswordAsync`) |
| `ConnectionWorkflowService.ConnectProfileAsync` after success | `:109` | Yes |
| `MainWindowViewModel.PersistProfileIfSavedAsync` | `:4519` (callers: FTP `:3713`, plugins `:3871`, workspaces `:4156`) | **No**, saves directly |
| `SessionTreeViewModel.MoveSessionToGroupAsync` (drag to another group) | `src/VelaShell.Presentation/ViewModels/SessionTreeViewModel.cs:443` | **No**, saves the tree-cache instance |
| `SessionTreeViewModel.DuplicateSelectedSessionAsync` (duplicate profile) | Same file `:625` | **No**, `Clone()` copies `Password` |
| Applying a Gist sync pull | `GistSyncService.cs:617-626` | **No**; when the incoming profile has no password, the locally stored one is copied back and saved directly |
| Session import | `SessionImportWriter.cs:52-61`, `:126` | Sets `RememberPassword` from the import result and saves directly |
| Saving in the profile dialog | `ConnectionProfileViewModel.SaveAsync` `:1167` → workflow | Yes |
| AI / plugin connection proposals | `MainWindow.axaml.cs:1106-1137` → profile dialog | Yes (through the dialog) |

`WithPersistablePasswordAsync` (`:121-146`) **strips only `Password`, not `PrivateKeyPassphrase`** (`:144`).

### 2.6 Plaintext residence in memory

| Location | How long it stays | Readers |
|---|---|---|
| `TerminalTabViewModel.Profile` (`:125`) | Tab lifetime | Reconnect, clone, diagnostics |
| `SshSession.ConnectionInfo` (`SshSession.cs:18`, assigned in `SshConnectionService.cs:185-188`) | Session lifetime, **including the password of every jump hop** | Reads only Host / Port / Username (`SessionsCapability.cs:36-38`, `SessionMetricsService.cs:139`, `RemoteProcessService.cs:125`, `MainWindowViewModel.cs:4312`) |
| The `PasswordCredential` in `TmdsSshClientWrapper._settings` (`:37`) | Session lifetime | Does not reconnect by itself (`AutoConnect = false`, DI `:299`) |
| `FtpConnectionPool.Info` (`:43`) + the factory closure (`FtpFileService.cs:44`) | Connection-pool lifetime, **used again whenever a new connection is opened on demand** | Connection pool |
| Session tree `_sessionCache`, command palette `_paletteProfiles`, tunnel panel `Servers` | Until the next refresh | Carry the hand-typed password after the dialog modified them in place |
| `SftpDocumentViewModel.Profile`, `ConnectingDocument.Profile`, `PluginWorkspaceDocument.Profile` | Document lifetime | — |

### 2.7 Other constraints

| Constraint | Basis | Impact on this design |
|---|---|---|
| `AuthMethod` is persisted as an ordinal, so values can only be appended | `AuthMethod.cs`, `ModelSerializationTests.AuthMethod_OrdinalValues_MustStayStable` | The "source" is a separate field; **no** `AuthMethod.PasswordManager` is added |
| A new `SessionProfile` field must also be added to `Clone()` and to the tests' `FullyPopulated()` | `tests/VelaShell.Core.Tests/Models/SessionProfileCloneTests.cs:25-61` | Otherwise the test does not go red when the default values happen to be equal |
| Unencrypted Gist sync clears `Password` / `PrivateKeyPassphrase` | `GistSyncService.cs:490-494` | A reference is not a secret and may roam with sync |
| Device-local fields are stripped on push and kept on pull by a pair of **mirror methods** | `ScrubDeviceLocalFields` `:681-696` / `PreserveDeviceLocalFields` `:699-711` | If local paths went into `AppSettings`, both sides would need additions (§10.1 suggests keeping them out entirely) |
| Configuration export clears only the proxy password | `SettingsViewModel.BuildExportJson` `:1334-1339` | Same as above |
| **The ConPTY local terminal inherits VelaShell's entire process environment** | `ConPtyShellStream.cs:223-226` (`lpEnvironment = IntPtr.Zero`) | **Never set process-level environment variables**, or `BW_SESSION` would leak into the user's shell |
| The codebase deliberately does not reference `Microsoft.Win32.SystemEvents` | `ConnectivityMonitor.cs:29-33` | "Clear the cache on screen lock / sleep" either reuses `ConnectivityMonitor.Resumed` or overturns that trade-off (§16 N6) |
| There is no general "start a process + timeout + capture output" wrapper | Each call site handles it itself; see `PluginProcessClient.cs:233-374` | `CliCredentialRunner` has to be written new |
| There is no executable signature verification anywhere | Update packages are only checked by SHA-256 | Authenticode verification cannot be copied from existing code; the 1Password app verifies `op.exe` itself (§8.1) |
| `SecureStringConvert` lives only in the App project | `src/VelaShell/Security/SecureStringConvert.cs` | Infrastructure cannot reach it; §16 N7 suggests moving it down to Core |
| `CryptographicOperations.ZeroMemory` is used nowhere in the codebase | — | Zeroing the unlock cache has to be written new |
| The app has trimming off and is not published with AOT | `VelaShell.csproj:40` | CLI JSON can be parsed with reflection-based System.Text.Json / `JsonDocument` |
| There are no layered-architecture tests; layering rests only on project references | — | New contracts go in Core, implementations in Infrastructure, UI in App |
| All services are singletons; multiple implementations are registered by "register the concrete type first, then forward" | `InfrastructureServiceCollectionExtensions.cs:70-81` | Providers are registered the same way |
| Interfaces implemented by the UI are resolved optionally through `sp.GetService<T>()` and fail closed when missing | `IHostKeyPrompt`: `App.axaml.cs:136`, DI `:125` | The unlock dialog plugs in the same way |
| Skipped integration tests use `Assert.Inconclusive`, no longer a `[SKIP]` line | `SshIntegrationTests.cs:90-117`, `CrossPlatformPublishTests.cs:92-105` | Real-CLI tests are gated the same way |
| Existing conclusions for the plugin route | [15 §2.4](../plugins/15-ecosystem-ideas.md), [12 §1](../plugins/12-security-threat-model.md) | The built-in approach does not go through PluginHost / RPC, so the threat model's promises still hold |
| Tmds.Ssh 0.24 supports fetching the password through a callback during authentication, and in-memory private keys | XML docs in the package: `PasswordCredential(PasswordPrompt)`, `PrivateKeyCredential(Func<CancellationToken, ValueTask<Key>>)` | Not used in this phase (private keys go through the agent, passwords are resolved up front); recorded for reference |

---

## 3. Existing defects found along the way

These four have nothing to do with credential managers and exist today. But this feature would amplify some of them, so **they should be fixed separately as phase 0a** (§16 N1).

### D1 · Passwords typed by hand can still be persisted when "Remember password" is unticked

**Reproduction path** (derived from the code, **to be confirmed by testing**):

1. A profile has `RememberPassword = false`, and no password is stored.
2. The user double-clicks it in the session tree. `RequiresCredentials` is true, the login dialog appears, and the user types the password and leaves "Remember" unticked.
3. The dialog writes the password into **the instance in the session-tree cache** (`MainWindow.axaml.cs:1533`). After a successful connection, the workflow strips the password according to the "Remember password" rule and then saves; that step is fine.
4. The user then **drags this profile into another group**. `MoveSessionToGroupAsync` saves the cached instance directly (`SessionTreeViewModel.cs:443`), and **the password is encrypted and written to the database**.
5. Likewise, "Duplicate profile" (`:625`) and `PersistProfileIfSavedAsync` after FTP / plugin certificate trust (`:4519`) also persist the hand-typed password.

**Impact**: it goes against the user's explicit choice. The credential is persisted silently, and nothing in the UI shows it.

**Fix** (together with D3):

- The login dialog **returns a copy** instead of modifying the instance passed in.
- The failure card's "Retry" closure captures the latest `current`, keeping today's "retry without typing again" experience.
- `PersistProfileIfSavedAsync` goes through the workflow's stripping logic.
- Once the dialog no longer modifies shared instances, the session-tree cache holds only values read from the repository (which are persisted already), so `MoveSessionToGroupAsync` and duplicating a profile naturally store nothing extra — **they need no individual changes**.

### D2 · `WithPersistablePasswordAsync` does not strip the private-key passphrase

- `:144` only says `stripped.Password = null`.
- The profile dialog's "Remember password" checkbox is shown only for password authentication (`ConnectionProfileView.axaml:531-546`, controlled by `ShowPasswordField`); the login dialog's private-key branch does not write back `RememberPassword` either. So the private-key passphrase is **in practice always persisted**.
- This may be intentional (the passphrase is entered on the profile page and should be saved by default), or an oversight. **Your confirmation of the semantics is needed** (§16 N2) before deciding whether to add the stripping or to state in a comment that "the passphrase is not governed by Remember password".

### D3 · The login dialog modifies shared instances in place

See §2.4. Besides causing D1, it leaves plaintext passwords scattered across the session-tree cache, the command-palette cache and the tunnel panel's list until the next refresh. The fix is the same as for D1.

### D4 · `SshSession.ConnectionInfo` holds every jump-host password for its whole lifetime

- While the session is alive, `SshSession.ConnectionInfo` keeps the complete `ConnectionInfo`, including the password and private-key passphrase of every hop, while its readers use only Host / Port / Username (§2.6).
- **Fix**: after `SshConnectionService.ConnectInternalAsync` hands the `ConnectionInfo` to the client factory, attach **a copy with the secrets removed** to `SshSession` (handling `JumpHost` recursively).
- This is low-risk hardening. Its relation to this feature: passwords resolved from a vault also enter `ConnectionInfo`, and without the hardening they would stay there for the whole session lifetime.

---

## 4. Decisions made

| # | Question | Final | Reason / impact |
|---|---|---|---|
| 1 | Which vendors first | **All three together** | The CLI runner and the unlock session component are done in one go in phase 1 |
| 2 | How private keys are fetched | **SSH agent** | Private keys never leave the vault process. **Wait for the upstream PR to be merged**, then a separate project; in this phase providers supply only username and password |
| 3 | Reference granularity | **Bind the whole item**, with an advanced option to override the username | Renaming the item or changing its password does not affect the profile |
| 4 | Unlock secret cache | **In-memory TTL, configurable default** | See §6 |
| 5 | On resolution failure | **Fall back to the dialog + notice bar** | See §5.8 |
| 6 | Reference and stored password coexisting | **Forbidden** | Enforced in the repository layer (§5.4) |
| 7 | Built in or plugin | **Built in** | The threat model is unchanged; no SDK release needed |
| 8 | Order relative to `IMasterKeyStore` | **Parallel** | The system keychain provider waits for its P/Invoke layer |

---

## 5. Architecture

### 5.1 Layers and placement

| Layer | What is added | Directory |
|---|---|---|
| Core | Contracts, models, exceptions, resolve options | `src/VelaShell.Core/Credentials/` (new directory) |
| Core | `SecureStringConvert` moved down (§16 N7) | `src/VelaShell.Core/Security/` |
| Infrastructure | `CliCredentialRunner`, the three providers, `CredentialResolver`, the unlock session cache, provider settings storage | `src/VelaShell.Infrastructure/Credentials/` |
| Infrastructure | DI registration | `InfrastructureServiceCollectionExtensions.cs`, a new `AddCredentialProviders` section |
| Presentation | `ConnectionWorkflowService` wired to the resolver | Existing file |
| App | Unlock dialog, item picker, settings page, changes to the profile page and the login dialog | `src/VelaShell/Views/`, `ViewModels/`, `Services/` |

The dependency direction is unchanged: Core ← Infrastructure ← App, Core ← Presentation ← App. Presentation **does not reference** Infrastructure; it depends only on `ICredentialResolver` in Core.

### 5.2 Contracts (`VelaShell.Core/Credentials/`)

```csharp
public interface ICredentialProvider
{
    /// "1password" / "bitwarden" / "keepassxc" / "os-keychain". Persisted in the reference; immutable once chosen.
    string Id { get; }

    /// Localized display name.
    string DisplayName { get; }

    /// Probe installation and sign-in state. **Must not trigger biometrics or an unlock dialog** (called as soon as the settings page opens, see §8.1).
    ValueTask<CredentialProviderStatus> GetStatusAsync(CancellationToken ct);

    /// Returns metadata only, never secrets. May need unlocking (interactive).
    Task<IReadOnlyList<CredentialItemSummary>> SearchAsync(string query, CredentialResolveOptions options, CancellationToken ct);

    /// Fetch username and password.
    Task<ResolvedCredential> ResolveAsync(CredentialReference reference, CredentialResolveOptions options, CancellationToken ct);

    /// Discard this provider's unlock cache (called by "Lock now" on the settings page, on TTL expiry, and when the network comes back). Does not call the vendors' CLI lock commands (see §8.2).
    void ForgetUnlock();
}

public enum CredentialProviderState { NotInstalled, NotSignedIn, Locked, Ready, Unknown }

public sealed record CredentialProviderStatus(
    CredentialProviderState State,
    string? Version,
    string? ExecutablePath,
    string? Detail);                    // localized extra explanation, shown on the settings page

public sealed record CredentialItemSummary(
    string ItemId,
    string? Scope,                      // 1Password: "<accountId>/<vaultId>"; KeePassXC: database alias; otherwise null
    string Title,
    string? Username,                   // null when the list does not provide it; filled in after selection (§8.1)
    string? Location,                   // 1Password vault name / KeePassXC group path / Bitwarden folder
    string? Url);

public sealed record ResolvedCredential(string? Username, string Password);

public sealed record CredentialResolveOptions(
    bool Interactive,                   // false: throw Locked straight away when unlocking is needed, never show a dialog (§5.7)
    string TargetDisplayName);          // the unlock dialog shows "to connect to xxx"; follows the "no username or port" display convention

public enum CredentialFailure
{
    NotInstalled, NotSignedIn, Locked, UnlockCancelled, UnlockFailed,
    NotFound, Ambiguous, MissingField, Timeout, Malformed, ProviderDisabled
}

public sealed class CredentialProviderException(
    CredentialFailure kind, string providerId, string? detail = null, Exception? inner = null)
    : Exception(detail, inner)
{
    public CredentialFailure Kind { get; } = kind;
    public string ProviderId { get; } = providerId;
}

/// Implemented by the UI layer (the same pattern as IHostKeyPrompt), fail-closed: returns null on an exception or when there is no main window.
public interface ICredentialUnlockPrompt
{
    /// The caller is responsible for zeroing the returned char[]. null = the user cancelled.
    Task<char[]?> RequestUnlockSecretAsync(CredentialUnlockRequest request, CancellationToken ct);
}

public sealed record CredentialUnlockRequest(
    string ProviderId,
    string ProviderDisplayName,
    string? TargetName,                 // KeePassXC database alias; for Bitwarden the account e-mail (available from bw status)
    string ConnectionDisplayName,
    bool PreviousAttemptFailed);        // shown again after a wrong secret; the dialog displays an error hint

/// The single entry point for the workflow and Infrastructure.
public interface ICredentialResolver
{
    /// Returns null when the profile has no reference. The result is not written back to the profile.
    Task<ResolvedCredential?> ResolveAsync(SessionProfile profile, CredentialResolveOptions options, CancellationToken ct);

    IReadOnlyList<ICredentialProvider> Providers { get; }
}
```

**About the callback's form**: `ICredentialUnlockPrompt` returns `char[]` rather than `SecureString`. The secret ultimately has to be written to the child process's stdin or environment, so `SecureString` would only add one more conversion here without adding real protection. Inside the dialog `SecurePasswordBox` is still used (it is backed by a `SecureString`), and its value is converted to `char[]` before being handed over.

### 5.3 Data model

`SessionProfile` gains:

```csharp
/// Credential source: when non-null the password is fetched from the password manager, and Password is never persisted (enforced by the repository).
public CredentialReference? CredentialSource { get; set; }
```

```csharp
public sealed record CredentialReference
{
    public required string ProviderId { get; init; }
    public required string ItemId { get; init; }
    public string? Scope { get; init; }
    public string? DisplayName { get; init; }        // snapshot of the item title at selection time, display only
    public string? DisplayUsername { get; init; }    // snapshot of the item username at selection time, display only
    public bool OverrideUsername { get; init; }      // true: SessionProfile.Username overrides the item's username
}
```

`ItemId` / `Scope` values per provider:

| Provider | `ItemId` | `Scope` | Notes |
|---|---|---|---|
| 1Password | item UUID | `<accountId>/<vaultId>` | The vendor recommends addressing by ID, avoiding name collisions and rate limiting (§8.1) |
| Bitwarden | item GUID | null | `bw` is signed in to only one account at a time |
| KeePassXC | Full entry path (`/group/title`) | Database alias (configured in settings) | The CLI **cannot address by UUID** (§8.3) |
| System keychain | TargetName on Windows | null | Phase 2 |

Implementation notes:

- `CredentialReference` is a record, so `Clone()` can simply assign the reference (it is immutable); add it to `SessionProfileCloneTests.FullyPopulated()` as well.
- JSON is persisted through `SonnetDbJson` (camelCase, nulls omitted when writing); old data without this field reads as null, so **zero migration**.
- The reference is **persisted in plaintext and travels with Gist sync**. The item-title snapshot, the username snapshot and the vault ID are metadata, not secrets; `PRIVACY.md` says so truthfully (§14).

### 5.4 The single gate for the persistence invariant: the repository

Add one block to `SonnetDbSessionRepository.Protect` (`:131-140`):

```csharp
if (copy.CredentialSource is not null)
{
    // Decision 6: a reference and a stored password never coexist. This lives in the repository, not the callers —
    // there are 7 save points (§2.5), 4 of which bypass the workflow; guarding each one would miss one sooner or later.
    copy.Password = null;
}
```

Reason: the repository is the **single exit** for every save. The direct saves listed in §2.5, copying the local password back during a sync pull (`GistSyncService.cs:623`), import and duplicating a profile are all covered by this one block automatically. `WithPersistablePasswordAsync` does not need the same check.

### 5.5 Resolution points: wiring the five construction points one by one

| # | Construction point | Change |
|---|---|---|
| 1 | `ConnectionWorkflowService.BuildChainAsync` | The constructor gains an optional `ICredentialResolver? credentialResolver = null`. For every hop: when `profile.CredentialSource != null` and the in-memory `Password` is empty, call the resolver and fill `ConnectionInfo.Username` / `Password`. **Nothing is written back to the profile**. `ConnectProfileAsync` and `TestConnectionAsync` gain a `CredentialResolveOptions` parameter (interactive by default) |
| 2 | FTP: `MainWindowViewModel.cs:3684` | Resolve before building `FtpConnectionInfo`. `FromProfile` gains an overload `FromProfile(SessionProfile, ResolvedCredential?)` |
| 3 | `PluginProtocolFileService.OpenSessionCoreAsync` | Inject the resolver at construction (inside Infrastructure, available through DI). Resolve before assembling `ProtocolConnectRequest` |
| 4 | `PluginTerminalShellStream` | Same as above. Its reconnect also goes through here |
| 5 | `PluginWorkspaceLauncher.OpenCoreAsync` | Same as above |

**Precedence rule**: **a non-empty in-memory `Password` takes precedence over the reference**.

- A password typed into the dialog lives in this connection's copy (once D1/D3 are fixed), so it naturally takes effect.
- Thanks to the invariant in §5.4, an instance read from the repository never has both a reference and a password.
- So this rule only matters for "typed by hand this time" and creates no ambiguity.

**Jump hops**: jump profiles are read from the repository; if one carries a reference, it is resolved with **the interactive mode of the current connection**.

- On resolution failure, **no login dialog is shown** (jump hosts never prompt today either); it ends as a connection error.
- The error message says which hop and which provider failed, for example "Jump host bastion-01: the Bitwarden vault is locked".

### 5.6 Resolver internals (`CredentialResolver`)

1. **Find the provider**: look it up by `ProviderId`; if it is not registered or is disabled in settings, throw `ProviderDisabled`.
2. **Per-attempt cache**: cache `ResolvedCredential` keyed by "connection attempt token + reference", and drop it when the attempt ends (success or giving up).
   - This way, within one attempt, neither an SSH jump chain nor a non-authentication error in a retry loop queries the vault twice.
   - In practice the token is created by the `TryConnect*` loop and passed down through `CredentialResolveOptions`; it could also be carried by `AsyncLocal` — pick whichever is simpler at implementation time.
3. **Serialize unlocking per provider**: one `SemaphoreSlim(1,1)` per provider.
   - When startup restore brings back 10 tabs, or several jump hops hit the same locked vault, the unlock dialog appears only once; the others wait and then reuse the same unlock session.
   - After the user cancels, the remaining queued requests **end immediately with `UnlockCancelled`** instead of prompting one by one.
4. **Resolved passwords are not cached**: across attempts (for example a manual reconnect an hour later) they are resolved again. As long as the vault's unlock session is still within its TTL, no dialog appears.
5. **Username rule**: when `OverrideUsername` is true, use `SessionProfile.Username`, otherwise the item's username. If both are empty, throw `MissingField`.

### 5.7 Interactive and non-interactive resolution

| Trigger | Mode | When the vault is locked |
|---|---|---|
| User-initiated connections: session tree, command palette, recent connections, "Connect" / "Test" in the profile dialog, clone tab, opening SFTP / FTP / plugin documents, external launch, restoring sessions at startup, opening a layout template (it goes through the same connection flows) | **Interactive** | Show the unlock dialog |
| Manual reconnect: Ctrl+R, Enter, the "Reconnect" button on the bubble (`:872`, `:1246/2259`, `:2782`) | **Interactive** | Show the unlock dialog |
| Automatic reconnect: the `OnTabDisconnected` timer (`:2812-2832`), `ReconnectAllAfterResume` (`:2694-2713`), tunnel `AutoReconnectAsync` | **Non-interactive** | Throw `Locked`; the tab shows the failure overlay with the text "The vault is locked. Press Ctrl+R to reconnect" |
| Connection diagnostics | Interactive (opened by the user) | Show the unlock dialog |
| Plugin `Sessions.OpenAsync` | Interactive (the host shows a confirmation dialog anyway) | Show the unlock dialog |

Changes: `ReconnectTabAsync` gains a `bool interactive` parameter passed in by each caller; `TunnelPanelViewModel.AutoReconnectAsync` passes `false`.

**Reason**: if 20 tabs reconnect automatically when the network comes back, interactive mode would immediately pop up an unlock dialog. The user may not even be at the computer, and coming back to an unlock request nobody knows the origin of is exactly the phishing concern of §15 R9.

### 5.8 Wiring the failure fallback (decision 5)

**All five retry loops change the same way**:

```csharp
catch (CredentialProviderException ex) when (InteractiveAuthenticator is not null)
{
    pendingNotice = CredentialNotice.From(ex);   // carried into the next dialog
    tab?.MarkConnectionFailed(...);              // SSH terminal path
    continue;                                    // attempt > 0 → the dialog is certain
}
```

- **Login dialog**: the `AuthenticationDialogViewModel` constructor gains `CredentialNotice? notice = null` and `bool hideRememberPassword = false`; the `InteractiveAuthenticator` delegate signature is extended accordingly (or an overload with options is added).
- **Paths without a dialog delegate** (for example the tunnel panel when it has none, or non-interactive reconnects): turn the exception into localized connection-error text and use the existing failure overlay or bubble.
- **A hand-typed password is valid for this time only**: for profiles with a reference, the login dialog hides "Remember password". Even if the dialog result comes back with `RememberPassword = true`, the repository invariant strips the password.
- **The server rejected the password from the vault** (`VelaSshAuthenticationException` and the like): the existing authentication-failure retry applies. The dialog's notice bar says "The server rejected the password from {provider}; the item may be out of date". The next attempt uses the password the user typed and **does not query the vault again** (precedence rule, §5.5).

**Notice bar contents**:

| `CredentialFailure` | Gist of the text | Extra action |
|---|---|---|
| `NotInstalled` | The {provider} command-line tool was not found | "Open settings" → jump to the credentials page |
| `ProviderDisabled` | {provider} is disabled in settings | "Open settings" |
| `NotSignedIn` | {provider} is not signed in (1Password: turn on the CLI integration in the app under "Settings → Developer"; Bitwarden: run `bw login` in a terminal first) | — |
| `Locked` / `UnlockCancelled` | The vault is not unlocked | "Retry unlock" |
| `UnlockFailed` | Wrong unlock secret | "Retry unlock" |
| `NotFound` | The item no longer exists (it may have been deleted, moved or renamed; KeePassXC addresses by path, so changing the title also loses it) | "Choose the item again" → opens the profile page |
| `Ambiguous` | Several items with the same name were found (KeePassXC) | "Choose the item again" |
| `MissingField` | The item lacks a username or password | — |
| `Timeout` | {provider} did not respond in time (1Password may be waiting for Windows Hello confirmation) | "Retry" |
| `Malformed` | {provider} returned a result that could not be understood, possibly a version incompatibility | "Open log folder" |

### 5.9 Where resolved values go in memory

| Carrier | Lifetime | Handling |
|---|---|---|
| `ConnectionInfo` → `SshSession.ConnectionInfo` | Session lifetime | Secrets removed once D4 is fixed |
| `ConnectionInfo` → `TmdsSshClientWrapper._settings` | Session lifetime | Inside Tmds.Ssh; kept as it is (the same as a hand-typed password) |
| `FtpConnectionInfo` → `FtpConnectionPool` | Connection-pool lifetime | **Kept as it is**: the pool needs the password when it opens new connections on demand. Re-resolving every time would trigger unlocking when connections are topped up in the background, which costs more than it gains |
| SDK request DTO → in-process plugin | Up to the plugin | The same trust level as a hand-typed password today; stated in the [12 threat model](../plugins/12-security-threat-model.md) |
| `tab.Profile` | Tab lifetime | The `tab.Profile` of a reference profile does not contain the resolved password, so a reconnect resolves again; the copy with a hand-typed password contains it (the same as today) |

### 5.10 Interaction with sync, import, export and other flows

| Flow | Behaviour | Code change needed? |
|---|---|---|
| Gist push (unencrypted) | `Password` cleared, the reference uploaded as usual | No |
| Gist push (end-to-end encrypted) | A reference profile has no `Password` in the store to begin with | No |
| Gist pull | When the remote profile carries a reference and this device has an old password, `:623` copies the password back and the repository invariant strips it right away | No |
| Gist pull onto a machine without that provider configured | Throws `NotInstalled` or `ProviderDisabled` at connect time and falls back to typing in the dialog | No |
| Session import | Importers do not produce references | No |
| Duplicate profile | `Clone()` copies the reference; both profiles point to the same item, as expected | No (just add it to `Clone`) |
| External launch | When the request carries no password the saved profile is reused, and a saved profile with a reference resolves normally | No; run a regression test |
| AI / plugin connection proposals | The proposal carries a plaintext password and is saved through the profile dialog; no reference is involved | No |
| Configuration export / import | If provider settings were kept in `AppSettings`, the local paths would be exported; kept outside `AppSettings` as §16 N4 suggests, they are unaffected | Depends on N4 |

### 5.11 Audit and logging

**Audit** (`IAuditLogService`, `Category = "security"`) records **only failures and unlock events**, so that not every connection writes one more entry:

| Action | When | Detail |
|---|---|---|
| `credential-failed` | Resolution throws `CredentialProviderException` | `{providerId} {kind}`, **without the item title** |
| `vault-unlocked` | Unlock succeeded | `{providerId} {targetName?}` |
| `vault-unlock-failed` | Wrong unlock secret | `{providerId}` |
| `vault-forgotten` | Cache cleared on TTL expiry, manual lock or network recovery | `{providerId} {reason}` |

`ProfileId` is set to the profile concerned. A failure to write the audit entry does not affect the connection, the same as `RecordHistoryAsync`.

**Diagnostic log**: `Trace.WriteLine("[Cred] …")`, recording only:

- the provider id, the subcommand name (`item get` / `unlock` and so on, **without arguments**), elapsed time, exit code and `CredentialFailure` kind;
- the first 200 characters of stderr, with e-mail addresses and paths replaced.

**stdout is never logged.**

### 5.12 Connection diagnostics

- `HasStoredCredentials` returns true for profiles with a reference.
- **Suggested improvement**: add a "Credential source" step before step 4 (authentication) of the diagnostics flow. It calls `GetStatusAsync` to report the provider state, then tries resolving in interactive mode, and on failure shows the text from the table in §5.8. That way a user investigating "can't connect" can tell whether the vault or the server is the problem. Step texts keep the `DiagSvc_` prefix.

---

## 6. Unlock session cache

### 6.1 What each provider caches

| Provider | What is cached | Notes |
|---|---|---|
| 1Password | **Nothing** | Authorization is managed by the desktop app: it expires after 10 minutes without use, 12 hours at most; locking the app revokes all authorization (vendor documentation, §8.1) |
| Bitwarden | The **session key** returned by `bw unlock` | The master password is used once; **the master password itself is not cached** |
| KeePassXC | The **database secret**, cached separately per database alias | The CLI has no notion of a session; every call needs the secret |
| System keychain | Nothing | The system handles it |

### 6.2 Rules

- **Setting** `UnlockCacheMinutes`: default **15**, `0` means no caching (ask every time), maximum **480** (the same order of magnitude as 1Password's 12 hours; anything longer makes no sense).
- **Absolute expiry**: counted from the moment of unlocking, with no sliding renewal. Under continuous use, sliding renewal amounts to never expiring.
- **When it is cleared**:
  - TTL expiry (one `Timer` per provider);
  - application exit;
  - the "Lock now" button on the settings page;
  - `bw status` returning `locked` (meaning the session key is no longer valid);
  - the KeePassXC secret being rejected;
  - **`ConnectivityMonitor.Resumed`**: on wake from sleep the network adapters reinitialize and raise this event (§16 N6).
- **Holding and zeroing**: `char[]` (secrets) or `byte[]` (the UTF-8 form of the session key); cleared with `CryptographicOperations.ZeroMemory`.
- **Honest limit**: writing into a child process's stdin or `ProcessStartInfo.Environment` inevitably creates a managed `string` copy, which stays on the heap until the GC runs. This is the same trade-off as at the `SecureStringConvert.ToPlaintext` hand-off; it is stated in a code comment rather than papered over.

---

## 7. CLI runner

`CliCredentialRunner` (`Infrastructure/Credentials/CliCredentialRunner.cs`) is the invocation wrapper shared by the three providers, and the place where the risk of the whole design is most concentrated.

### 7.1 Interface draft

```csharp
internal sealed record CliInvocation(
    string ExecutablePath,
    IReadOnlyList<string> Arguments,                          // through ArgumentList, never string concatenation
    IReadOnlyDictionary<string, string?>? Environment,        // applies to the child process only; a null value deletes the inherited variable of that name
    ReadOnlyMemory<char> StdinSecret,                         // optional; stdin is closed right after writing
    TimeSpan Timeout,
    int MaxStdoutBytes = 4 * 1024 * 1024);

internal sealed record CliResult(int ExitCode, byte[] Stdout, string StderrRedacted, TimeSpan Elapsed);

internal sealed class CliCredentialRunner
{
    Task<CliResult> RunAsync(CliInvocation invocation, CancellationToken ct);
}
```

`Stdout` is returned as `byte[]` rather than `string`: the caller can zero it as soon as parsing is done, and it can be handed straight to `JsonDocument.Parse(ReadOnlyMemory<byte>)`.

### 7.2 Hard rules

| # | Rule | Reason / basis |
|---|---|---|
| 1 | The executable is **the absolute path saved in settings**; PATH is not searched at call time | Prevents PATH hijacking |
| 2 | Secrets **never go into command-line arguments** | Arguments show up in the process list |
| 3 | Secrets are passed through **stdin** (KeePassXC) or **written only into the child process's environment** (Bitwarden's `--passwordenv` and `BW_SESSION`) | Same as above |
| 4 | **Never call** `Environment.SetEnvironmentVariable` | The ConPTY local terminal inherits VelaShell's entire process environment (`ConPtyShellStream.cs:223-226`) |
| 5 | Explicitly delete inherited sensitive variables: Bitwarden calls delete `BW_SESSION` (and then write VelaShell's own when needed); 1Password calls **leave** `OP_ACCOUNT` / `OP_BIOMETRIC_UNLOCK_ENABLED` / `OP_SERVICE_ACCOUNT_TOKEN` **untouched** | When the user starts VelaShell from a shell that has `BW_SESSION`, that session should not be borrowed silently; 1Password's environment variables are the user's explicit configuration and should be respected |
| 6 | `UseShellExecute = false`, `CreateNoWindow = true`, stdin / stdout / stderr redirected | Avoids a console window flashing up on Windows |
| 7 | `StandardOutputEncoding = StandardErrorEncoding = UTF8` (no BOM); stdin is written as UTF-8 too | Item titles and secrets may contain non-ASCII characters, which the Windows console's default code page would garble (the actual output encoding of the three CLIs on Windows is **to be tested**) |
| 8 | On timeout, `process.Kill(entireProcessTree: true)`; the cancellation token is handled the same way | See `PluginProcessClient.TryKill` (`:361-374`) |
| 9 | When stdout reaches `MaxStdoutBytes`, truncate, kill the process and report `Malformed` | Keeps abnormal output from exhausting memory |
| 10 | Only the first 200 characters of stderr are kept, and e-mail addresses and paths are replaced before anything reaches the log | §5.11 |
| 11 | **stdout never goes into the log or into exception messages** | It may contain passwords |
| 12 | Everything runs on the thread pool, never on the UI thread | 1Password's Windows Hello window is shown by its own app; VelaShell only awaits |
| 13 | CLI calls for the same provider run serially (sharing the lock with the unlock serialization of §5.6, or using a separate one) | The behaviour of `bw` and `keepassxc-cli` accessing the same data directory or database concurrently is **to be tested**; serial is the safest |
| 14 | Default timeouts: 90 seconds interactive (leaving time for Windows Hello / a YubiKey touch), 20 seconds non-interactive, 5 seconds for status probes | Can be overridden in the provider implementation |

### 7.3 Locating the executable

- **First choice** is the absolute path saved in settings; if the file does not exist, report `NotInstalled`.
- **Auto-detection** (the "Auto-detect" button on the settings page, or run once the first time the settings page is opened):
  - Search PATH, following `LocalShellCatalog.FindOnPath` (`:53-76`). That one only works on Windows and does not handle PATHEXT, so a cross-platform version has to be written here (on Windows try `.exe` / `.cmd` in turn; `bw` installed through npm is `bw.cmd`, **to be tested**).
  - Then check each vendor's usual install directories (hard-coded only after **testing**, not from memory).
  - The detection result is written back to settings and shown, and **is enabled only after the user confirms** — never silently.
- **Signature verification (optional, not required for v1)**:
  - The codebase has no precedent for Authenticode verification.
  - On Windows, the 1Password app itself verifies the Authenticode signature of the connecting `op.exe` (vendor documentation), so for 1Password a check on the VelaShell side would duplicate work.
  - Bitwarden / KeePassXC have no comparable mechanism. If it is done, use `WinVerifyTrust` on Windows; skip other platforms for now.

### 7.4 JSON parsing

- Use `JsonDocument` to pick fields as needed (following the DOM style of `HttpPluginMarketClient.cs:104,229`), **without defining strongly typed models**, so fields added to or removed from CLI output are tolerated.
- After parsing, zero the stdout byte array.

---

## 8. Specifications of the three providers

### 8.1 1Password (`op` v2)

**Verified facts**

- **App integration**: turned on in the 1Password app under "Settings → Developer → Integrate with 1Password CLI"; supported on Mac / Windows / Linux, Linux needs PolKit; Windows uses Windows Hello.
- **Authorization lifetime**: "expires after 10 minutes of inactivity", "hard limit of 12 hours", "automatically renewed on every use".
- **Authorization scope**: on macOS / Linux limited to one terminal session, extending to sub-shells of that window; **"On Windows, commands executed in a sub-shell require separate authorization"**.
- **App locked**: "When the 1Password app is locked, all prior authorization is revoked".
- **Communication on Windows**: a named pipe; the app verifies the Authenticode signature of the connecting process's executable.
- **`op item get`**: `op item get [{ <itemName> | <itemID> | <shareLink> | - }] [flags]`, supporting `--fields label=username,label=password` and `type=` filters, `--vault`, `--reveal` (shows concealed fields in human-readable output), `--otp`, `--include-archive`. **The vendor recommends addressing by ID**, which avoids rate limiting and same-name ambiguity.
- **`op item list`**: supports `--vault`, `--categories`, `--tags`, `--favorite`, `--long`, `--include-archive`. **The structure of the JSON fields is not documented**.
- **Multiple accounts**: the `--account` flag, or the `OP_ACCOUNT` environment variable.
- **`op whoami`**: "returns an error when there is no authenticated account".
- **`OP_BIOMETRIC_UNLOCK_ENABLED`**: switches the app integration on or off.

**Design**

| Operation | Command | Notes |
|---|---|---|
| Status probe | `op --version`; then `op account list --format json` | **Do not call `op whoami` or `item` commands**, so that opening the settings page does not raise Windows Hello. Whether `account list` itself triggers authorization is **to be tested**; if it does, the status probe stops at `--version` and the sign-in state is shown as `Unknown` |
| Search | `op item list --categories Login --format json [--account <id>] [--vault <id>]` | Triggers authorization (acceptable, since the user opened the picker). Results are cached for this picker session |
| Fill in the username | After the user selects an item, run `op item get <itemId> --vault <vaultId> --account <accountId> --fields label=username --format json` | Whether the list JSON contains the username is undocumented, so it is not relied on |
| Resolve | `op item get <itemId> --vault <vaultId> --account <accountId> --format json` | Fetch the full item JSON and **pick fields by their `purpose` first (`USERNAME` / `PASSWORD`)**, falling back to `label` if not found. That way the right values are taken even if the user renamed the field labels. Whether the `purpose` field exists is **to be tested**; whether concealed fields in JSON output need `--reveal` is **to be tested** (community reports say the JSON contains the values directly); until tested, always pass `--reveal` |

**Failure mapping** (matching rules for the error texts to be completed **after testing**):

| Situation | `CredentialFailure` |
|---|---|
| The executable does not exist | `NotInstalled` |
| No account, or the app integration is off | `NotSignedIn` |
| The user clicked Cancel in the Windows Hello prompt | `UnlockCancelled` |
| The item does not exist or is archived | `NotFound` |
| Timeout | `Timeout` |

**⚠️ First test item: authorization scope on Windows**

The vendor only says "sub-shells require separate authorization", and does not say how an `op` launched by a GUI process counts. Possible outcomes:

- (a) The VelaShell process counts as the "terminal session", and several calls within 10 minutes need only one authorization — the ideal case;
- (b) Every call to `op` needs one Windows Hello — one connection must then be squeezed down to **a single call** to `op` (filling in the username and resolving merged into the same `item get`), and the picker's search must cache its results too;
- (c) Something else.

Write the test result back into this section and decide from it whether calls need merging.

**Pitfalls to avoid**

- If the user's environment has `OP_SERVICE_ACCOUNT_TOKEN`, `op` switches to the service account and bypasses the app integration. This is the user's own configuration; the status probe should recognize it and show "service account mode".
- 1Password's SSH agent belongs to the private-key track and is unrelated to this phase.

Sources: [item command reference](https://www.1password.dev/cli/reference/management-commands/item) · [App integration](https://www.1password.dev/cli/app-integration) · [App integration security model](https://www.1password.dev/cli/biometric-security) · [whoami](https://www.1password.dev/cli/reference/commands/whoami) · [CLI release notes](https://releases.1password.com/developers/cli/)

### 8.2 Bitwarden (`bw`)

**Verified facts**

- **`bw status`** returns `{ serverUrl, lastSync, userEmail, userId, status }`. `status` is `"unlocked"` / `"locked"` / `"unauthenticated"`. Before answering, the CLI tries an automatic unlock once; a failure does not throw.
- **Global options**: `--raw` (raw output), `--response` (JSON wrapper), `--quiet`, `--nointeraction` ("Do not prompt for interactive user input"; internally sets `BW_NOINTERACTION=true`), `--session <key>` (internally equivalent to setting `BW_SESSION`), `--cleanexit`, `--pretty`.
- **`bw unlock [password]`** is for "Unlock the vault and return a new session key", and supports `--check`, `--passwordenv <var>`, `--passwordfile <path>`. The session key stays valid until `bw lock` or `bw logout`.
- **`bw lock`** is for "Lock the vault and destroy active session keys".
- **`bw list items`** supports `--search`, `--folderid`, `--url`. **The output is full items, including `login.password`**.
- **`bw get item <id>`** returns the full item JSON; there are also `bw get password <id>` / `bw get username <id>`.
- **`bw sync`** (`-f` forces a full sync, `--last` shows the last sync time); **`bw config server <url>`**.
- **Sign-in methods**: e-mail and password (with two-step login), `--apikey` (`BW_CLIENTID` / `BW_CLIENTSECRET`), `--sso`.
- **Installation**: native executables (Windows / macOS / Linux x64), npm, snap, Chocolatey. It is a Node.js application.

**Design**

| Operation | Command | Notes |
|---|---|---|
| Common convention | Every call carries `--nointeraction` | With stdin redirected, a CLI waiting for interactive input would hang until the timeout |
| Status probe | `bw status --nointeraction` (with `BW_SESSION` deleted from the child process's environment) | Map `status` to `NotSignedIn` / `Locked` / `Ready`; `userEmail` is used as the unlock dialog's `TargetName` |
| Unlock | `bw unlock --passwordenv VELA_BW_PW --raw --nointeraction`, with `VELA_BW_PW` written only into the child process's environment | stdout is the session key, which goes into the cache of §6. **`--passwordfile` is not used** (it would touch the disk) |
| Search | `bw list items --search <q> --nointeraction`, environment variable `BW_SESSION=<key>` | ⚠️ **The output contains passwords**: parse it as a stream into `CredentialItemSummary` (`id`, `name`, `login.username`, `login.uris[0].uri`, `folderId`), discard it right away and zero the stdout byte array. **Called only once per picker session**; filtering is local after that |
| Resolve | `bw get item <id> --nointeraction`, `BW_SESSION=<key>` | Take `login.username` / `login.password`; report `MissingField` when the item type is not login |
| Sync | `bw sync --nointeraction` | Triggered manually by the "Sync" button in the picker; **no automatic sync** |
| Lock | **`bw lock` is not called** | It destroys every session in the CLI's data directory, invalidating even the `BW_SESSION` in the user's own terminal. VelaShell only discards the session key in memory |

**Failure mapping**:

| Situation | `CredentialFailure` |
|---|---|
| `unauthenticated` | `NotSignedIn` (signing in is out of scope for this phase; the user is told to run `bw login` in a terminal) |
| `locked`, in non-interactive mode | `Locked` |
| Wrong unlock secret (error text **to be tested**) | `UnlockFailed`; in interactive mode prompt again, at most 3 times |
| `get item` finds nothing | `NotFound` |

**Pitfalls to avoid**

- A self-hosted server is configured by the user with `bw config server`; VelaShell does not do it for them.
- Installed through npm, the entry point on Windows is `bw.cmd`, which has to go through `cmd.exe` (**to be tested**). If so, argument escaping needs extra care, or users should be advised to install the native build instead.
- A Node application pays a startup cost every time; the exact time is **to be tested**. The picker's search needs debouncing and should reuse results.

Sources: [Bitwarden CLI help](https://bitwarden.com/help/cli/) · [program.ts](https://github.com/bitwarden/clients/blob/main/apps/cli/src/program.ts) · [status.command.ts](https://github.com/bitwarden/clients/blob/main/apps/cli/src/commands/status.command.ts)

### 8.3 KeePassXC (`keepassxc-cli`)

**Verified facts**

- **Reading the secret**: `Utils::getPassword` uses `QTextStream` to **read one line from stdin**, toggling echo before and after; **piped input works too**. The prompt "Enter password to unlock %1: " is written to **stderr**, or to the null device with `-q`.
- **Database options**: `-k/--key-file <path>`, `--no-password`, `-y/--yubikey <slot[:serial]>` (slot can only be 1 or 2), `-q/--quiet`.
- **`show`**: supports `-a/--attributes <name>...`, `-s/--show-protected`, `--all`, `-t/--totp`, `--show-attachments`.
  - With `-a` it **prints one value per line**;
  - attribute names are **case-sensitive and support partial matching** — an ambiguous one reports "ERROR: attribute %1 is ambiguous, it matches %2.";
  - without `-a`, protected attributes are shown as `PROTECTED`.
- **Entry addressing**: `Group::findEntryByPath` **accepts only paths, not UUIDs**. When titles collide it **returns the first match**. When nothing is found it reports "Could not find entry with path %1." with exit code `EXIT_FAILURE`.
- **`ls`** supports `-R/--recursive` and `-f/--flatten`; the output format of **`search <database> <term>`** is undocumented; **`open`** enters an interactive shell mode.
- **There is no JSON output**.

**Design**

| Operation | Command | Notes |
|---|---|---|
| Status probe | `keepassxc-cli --version`; then check that each database file in settings exists and is readable | The database is not opened |
| Search | `keepassxc-cli ls -R -f -q [-k <keyfile>] [-y <slot>] <db>`, with the secret passed through stdin | Prints flattened entry paths (groups end with `/`; the exact format is **to be tested**), filtered locally. The results **contain only paths, no secrets**, and are cached for that database's unlock session |
| Fill in the username | After selection, run `show -q -a UserName <db> <path>` | Or show no usernames during search, to reduce the number of KDF runs (see "Performance" below) |
| Resolve | `keepassxc-cli show -q -s -a UserName -a Password [-k] [-y] <db> <entryPath>`, with the secret passed through stdin | The first line is the username, and **the remaining lines joined together are the password** (in case the password itself contains a newline). The behaviour of `-s` together with `-a` across versions is **to be tested** |
| Duplicate check at selection | When the same path appears more than once in the `ls` result, the picker warns "Entries with the same name exist; another entry may be picked" and suggests renaming it in KeePassXC first | Because the CLI takes the first match by path |

**Failure mapping**:

| Situation | `CredentialFailure` |
|---|---|
| The database file does not exist or is unreadable | `NotInstalled`, with the text changed to "Database file unavailable" |
| Opening the database failed (a wrong secret gives only a generic open error; the exact text is **to be tested**) | `UnlockFailed` |
| "Could not find entry with path" | `NotFound` |
| "is ambiguous" | `Malformed`, with a hint that an attribute name collides |
| Timed out waiting for a YubiKey touch | `Timeout` |

**Performance: every call re-derives the key**

Every CLI call runs the KDF again. The default Argon2 parameters of KeePass databases are **deliberately expensive**; the exact time depends on the database settings and is **to be tested**. So one "search + fill in the username + resolve" runs the KDF three times. Two options:

- **v1 (recommended)**: call per operation, but merge as much as possible — resolving calls `show` only once (fetching both attributes), the picker calls `ls` only once, and the username is fetched only for the selected entry.
- **Later optimisation (§16 N8)**: keep one interactive `keepassxc-cli open` process per unlock session and send commands through stdin, running the KDF only once. The price is that the database stays unlocked in the child process, interactive-mode output is harder to parse, and the process stays resident. Do it only if v1's testing proves it slow enough to hurt.

Sources: [keepassxc-cli manual](https://github.com/keepassxreboot/keepassxc/blob/develop/docs/man/keepassxc-cli.1.adoc) · [Utils.cpp](https://github.com/keepassxreboot/keepassxc/blob/develop/src/cli/Utils.cpp) · [Show.cpp](https://github.com/keepassxreboot/keepassxc/blob/develop/src/cli/Show.cpp) · [Group.cpp](https://github.com/keepassxreboot/keepassxc/blob/develop/src/core/Group.cpp)

---

## 9. System keychain provider

- **Phase 2**, depending on the `IMasterKeyStore` P/Invoke layer from phases 0/1 of the [keychain research](system-keychain-and-sudo-credential-fill-feasibility-research.md).
- **Windows first**: `CredEnumerateW` for search (filtered by TargetName), `CredReadW` to read `CRED_TYPE_GENERIC` entries; `ItemId` is the TargetName, the username comes from `UserName` and the password from `CredentialBlob` (UTF-16; whether every writer uses UTF-16 is **to be tested**).
- No CLI runner and no unlock cache are needed.
- Implementation details for macOS (`SecItemCopyMatching`) and Linux (libsecret) are settled together with the platform layer in §2 of the research document.

---

## 10. Settings and UI

### 10.1 Where the settings live

**Suggestion (§16 N4): keep provider settings out of `AppSettings` and store them as a separate local document**, modelled on `SyncSettings` (`Core/Sync/SyncModels.cs`).

| | In `AppSettings` | **Separate local document (recommended)** |
|---|---|---|
| Device-local fields such as executable and database paths | Must be added to both `ScrubDeviceLocalFields` and `PreserveDeviceLocalFields`; missing either side makes paths leak across devices | Never takes part in sync, by construction |
| Configuration export | Paths travel with the export file | Not involved |
| Loading and saving on the settings page | Must be added in both `ApplyToViewModel` and `SaveAsync` (`SettingsViewModel.cs:1454-1495`, `:1674-1712`) | The settings page VM reads and writes it itself, as `SyncPage` does |
| TTL preference roaming across devices | Yes | No (minor impact) |

Document structure (SonnetDB document collection `app_config`, id `credential_providers`; collection and naming conventions **to be confirmed at implementation**):

```
CredentialProviderSettings
├─ UnlockCacheMinutes : int = 15          (0 ~ 480, clamped in Normalize)
├─ OnePassword  { Enabled, ExecutablePath, DefaultAccountId? }
├─ Bitwarden    { Enabled, ExecutablePath }
└─ KeePassXc    { Enabled, ExecutablePath,
                  Databases: [{ Alias, Path, KeyFilePath?, YubiKeySlot? }] }
```

- Once a database **alias** is used by a reference, renaming it must warn "N profiles reference this alias".
- Deleting a database entry requires confirmation first.
- Every field must have a runtime reader — no new "stored but has no effect" switches; update [settings-audit.md](settings-audit.md) accordingly.

### 10.2 Settings page

**Suggestion (§16 N5): add a separate settings page "Credential managers"** rather than squeezing it into "General → Security and Privacy". The reason is the amount of content: three provider rows, plus the KeePassXC database list, plus the TTL and "Lock now".

What a new settings page needs to change (following the existing pattern):

1. Add a member to `enum SettingsSectionKey` (`SettingsViewModel.cs:51-88`; **the enum order is the navigation order**, so it should go after "Security Audit");
2. Add an entry to `BuildSections()` (`:495-545`), with the navigation text key `SetVm_SectionCredentials`;
3. Add a switch branch to `SettingsPageHost.Create` (`SettingsPageHost.cs:52-69`);
4. Update the hard-coded page order in `tests/VelaShell.Tests/ViewModels/SettingsSectionKeyTests.cs:26-47`;
5. The following tests cover the new page automatically and must pass: `SettingsLazyPagesUiTests` (page count equals enum count), `SettingsBindingErrorTests` (no binding errors), `AccessibleNameTests` (every button needs an accessible description).

**Page layout**:

```
Credential managers
  Read connection passwords from a password manager. VelaShell does not store these passwords, only references to the items.

  1Password                         ● Ready · 2.x.x         [Enabled ◯]
    CLI path         C:\…\op.exe           [Browse…] [Auto-detect]
    Default account  ▾ my.1password.com
  Bitwarden                         ● Locked · 202x.x.x     [Enabled ◯]
    CLI path         …                     [Browse…] [Auto-detect]   [Lock now]
  KeePassXC                         ● Installed · 2.7.x     [Enabled ◯]
    CLI path         …                     [Browse…] [Auto-detect]   [Lock now]
    Databases
    ┌ Alias ────┬ Path ──────────────┬ Key file ┬ YubiKey ┬─────────┐
    │ work      │ D:\kp\work.kdbx    │ —        │ 2       │ [Delete]│
    └───────────┴────────────────────┴──────────┴─────────┴─────────┘
    [Add database…]

  Keep unlocked for  [ 15 ] minutes (0 = ask every time)
```

- **Status probe**: runs asynchronously when the page is entered, written the same way as the About page's update check `UpdateStatus` (`SettingsViewModel.cs:779-855`). **No probing at application startup**.
- **Status display**: use `StatusDot` (DESIGN.md §5.2); do not add colour literals.
- **Database table**: follow the `Border.table` + `ItemsControl` of `KeyManagementPage.axaml:16-88`. There is currently **no** precedent for an editable-in-place list, so "Add database" is a small pop-up form, not in-table editing.
- **File picking**: use the `StorageProvider.OpenFilePickerAsync` pattern (`AppearanceSettingsPage.axaml.cs:17-37`) with a `.kdbx` filter.

### 10.3 Connection profile page

The changes are in `ConnectionProfileView.axaml` and `ConnectionProfileViewModel.cs`:

| Area | Change |
|---|---|
| Password area (`:469-492`, controlled by `ShowPasswordField`) | Add a "Password source" segmented choice above it: **Enter manually / Password manager**. The segments reuse the `seg-tab` style of `AuthenticationDialogView.axaml:60-74` (it would be worth moving that style group into the shared theme along the way, §16 N10) |
| With "Password manager" selected | Hide the password box and the "Remember password" checkbox (`:531-546`); show an item card: provider icon, item-title snapshot, username snapshot, a "Change item" button |
| Advanced area (`:566-674`, `IsAdvancedVisible`) | Add a checkbox "Use this profile's username instead of the item's". Only when it is ticked is the username box in the basic area editable; unticked, the item's username snapshot is shown (read-only) |
| Advanced-area badge `AdvancedBadge` | Overriding the username counts as one advanced item |
| Switching the source | Switching from "manual" to "password manager" while a password exists asks for confirmation with `MessageDialog.ConfirmAsync` ("The saved password will be cleared"); switching from "password manager" back to "manual" clears the reference |
| `BuildProfile()` (`:1265-1317`) | Writes `CredentialSource`; with the password manager as the source, `Password = null` |
| Loading (around `:143`) | Read `existing.CredentialSource` to decide the initial state of the segmented choice |
| Only for password authentication | Private-key / certificate authentication (`:496-529`) does not show the source choice (decision 2) |
| Plugin protocols | A plugin may relabel the password (for example S3's Secret Access Key); the source choice is shown as usual, and the label follows `PasswordLabel` |
| FTP anonymous login | The whole password area is hidden, as today |
| Child dialog | "Change item" opens the picker through a delegate (following the injection style of `CopyToClipboard`, `.axaml.cs:226`) |

### 10.4 Item picker (new dialog)

```
 Choose a credential item                                     ✕
 ┌──────────────┐ ┌───────────────────────────────────┐
 │ 1Password  ▾ │ │ 🔍 Search title, username, URL    │  [Sync]
 └──────────────┘ └───────────────────────────────────┘
 ┌─────────────────────────────────────────────────────┐
 │ prod-db-root          root         Private / DB     │
 │ bastion-01            ops          Infra            │
 │ ⚠ web (2 same name)   deploy       /Servers         │   ← KeePassXC duplicate-name warning
 └─────────────────────────────────────────────────────┘
                                          [Cancel]  [Select]
```

- The **search box** follows `CommandPaletteView.axaml:80-100`, the **table** follows `KeyManagementPage`.
- The provider dropdown lists only providers that are **enabled and whose executable has been detected**; once KeePassXC is selected, a database dropdown appears as well.
- **On first open**, the list is fetched once in that provider's way (1Password `item list`, Bitwarden `list items`, KeePassXC `ls`); after that it is **filtered locally**, with a 150 ms input debounce.
- The "Sync" button is shown only for Bitwarden.
- The picker is a dialog the user opens explicitly, so it **may take focus**; this does not conflict with the research document's §4 rule that "the hint bar never steals focus".
- When unlocking is needed, the unlock dialog is shown through `ICredentialUnlockPrompt` — the same one used when connecting.
- The return value is a `CredentialReference` (with title and username snapshots); cancelling returns null.

### 10.5 Login dialog notice bar

- **Position**: below the information bar of `AuthenticationDialogView.axaml:111-133`, or replacing the information bar's second line.
- **Style**: follow the warning bar of `HostKeyPromptView.axaml:113-139` (`VelaBgActive` background + `triangle-alert` icon + `VelaWarning` text). DESIGN.md has no `*Dim` warning colour, so **do not invent colours**.
- **Contents**: the text from the table in §5.8, plus at most one extra action button; the button needs an accessible description.
- **Profiles with a reference**: hide the "Remember password" controls (the AES badge at `:268-280` is hidden too).
- **Suggested improvement (§16 N10)**: the codebase currently has five or six notice bars, each written its own way (the report cited in §2.7); this is a chance to extract one shared `Border.notice` style and add it to DESIGN.md. Skipping it does not affect this feature.

### 10.6 Unlock dialog (new dialog)

- **The three pieces** follow `IHostKeyPrompt` → `HostKeyPromptDialogService` (`src/VelaShell/Services/HostKeyPromptDialogService.cs:15-49`) → `HostKeyPromptView`:
  - `CredentialUnlockDialogService : ICredentialUnlockPrompt`, switching to the UI thread with `Dispatcher.UIThread.InvokeAsync`;
  - the main window, taken from `IClassicDesktopStyleApplicationLifetime.MainWindow`, is the owner;
  - **it returns null when there is no main window or an exception occurs** (fail-closed).
- **Registration**: add `.AddSingleton<ICredentialUnlockPrompt, CredentialUnlockDialogService>()` near `App.axaml.cs:136`; Infrastructure resolves it optionally with `sp.GetService<ICredentialUnlockPrompt>()`. When it cannot be resolved, interactive resolution always reports `Locked`.
- **Input box**: reuse the `SecurePasswordBox` + `input-affix` + show-password button of `AuthenticationDialogView.axaml:231-255`.
- **Dialog contents** (anti-phishing, §15 R9):
  - title "Unlock {provider}";
  - target: the database alias or account e-mail;
  - reason: "To connect to {connection display name}" (**without username or port**, following the `ProfileDisplayName` convention);
  - an error hint when the previous secret was wrong.
- **When it appears**: **only within an operation chain the user started** (§5.7); it never pops up by itself in the background.
- **Behaviour**: Esc equals Cancel; on close, focus returns to the window that invoked it.

### 10.7 Localization

| Prefix | Use | Examples |
|---|---|---|
| `Cred_` | Service-layer texts, failure notice bars | `Cred_Failure_Locked`, `Cred_Failure_NotFound` |
| `CredUnlock_` | Unlock dialog | `CredUnlock_Title`, `CredUnlock_Reason` |
| `CredPicker_` | Item picker | `CredPicker_SearchPlaceholder`, `CredPicker_DuplicateWarning` |
| `SetCred_` | Settings page | `SetCred_UnlockMinutes`, `SetCred_LockNow` |
| `SetVm_SectionCredentials` | Settings navigation | — |
| `Profile_CredSource*` | Profile page | `Profile_CredSourceManual`, `Profile_CredSourceVault`, `Profile_CredOverrideUsername` |
| `Auth_CredNotice*` | Login dialog | `Auth_CredNoticeRejected` |
| `DiagSvc_Cred*` | The new connection-diagnostics step | — |

Notes:

- **All five resx files must be completed together** (`Strings` / `zh-Hans` / `zh-Hant` / `ja` / `ko`); `LocalizationTests.AllCultures_HaveIdenticalKeySets` checks this.
- The mapping from `CredentialFailure` to text keys is **written as an explicit `switch` with one literal key per branch**, not composed as `$"Cred_Failure_{kind}"`. Otherwise `UnusedLocalizedKeyTests` would flag those keys as orphans, and its `ComposedKeyPrefixes` (`UnusedLocalizedKeyTests.cs:31-35`) would have to be changed too.
- Keys used through `Strings.Format("Key", …)` are **not** checked by `LocalizedKeyUsageTests`; misspellings have to be caught by unit tests.

---

## 11. Out of scope

| Item | Why not in this phase | Where it goes |
|---|---|---|
| Fetching private keys / certificates from the vault | Decision 2: through the SSH agent | The agent feature becomes its own project once the upstream PR is merged; `plan.md` §17-A is revised at that point (it deliberately excluded `SshAgentCredentials`) |
| Fetching the passphrase of a local private-key file from the vault | Overlaps with the private-keys-through-the-agent direction; the need is unconfirmed | §17 |
| Credential providers as plugins | Decision 7 | Deferred per [15 §2.4](../plugins/15-ecosystem-ideas.md) |
| Bitwarden sign-in and two-step login | Complex interaction, and a one-time operation the user can finish in a terminal | Re-evaluate when there is demand |
| Filling at sudo prompts | Phase two of the research document | Reuses this phase's `SearchAsync` / `ResolveAsync` |
| TOTP / verification codes | Tied to keyboard-interactive authentication; a separate track | Re-evaluate when there is demand |
| Background checks for changed vault items | Would create periodic outbound calls, against the stance of `PRIVACY.md` | Not done; the `NotFound` notice bar is the safety net |
| Fetching the proxy password (`ProxyOptions.Password`) from the vault | Not part of the connection profile; a different save path | Re-evaluate when there is demand |
| Listening for system screen-lock events | The codebase deliberately does not reference `SystemEvents` (§16 N6) | Replaced by `ConnectivityMonitor.Resumed` and the TTL |

---

## 12. Phasing and task list

### Phase 0a · Fix existing defects (separate PR, best merged first)

> v3: D1 / D3 were fixed together with #550 (no separate PR — the "hand-typed entry is used only this once" of shared credentials is built right on top of it); D2 / D4 are still not done.

- [x] **D1 / D3**: `PromptCredentialsCoreAsync` in `MainWindow.axaml.cs` writes into `profile.Clone()` and returns the copy (v3: the conversion was extracted into `SharedCredentialPrompt.Apply`, unit-testable without a window)
- [x] **D1 / D3**: the "Retry" closures on the failure cards of the four document paths (`MainWindowViewModel.cs:3492,3657,3787,4069`) capture the latest `current` (v3: each pane of a dual-pane document captures its own as well)
- [x] **D1 / D3**: the semantics of `tab.Profile = current` (`:3286`) in the SSH terminal loop stay unchanged, with a comment added
- [x] **D1**: `PersistProfileIfSavedAsync` (`:4509-4526`) saves through the workflow's stripping logic. Either add `IConnectionWorkflowService.SaveProfileMetadataAsync` or reuse `SaveProfileAsync` (v3: reuses `SaveProfileAsync`)
- [ ] **D2**: following the outcome of §16 N2, add passphrase stripping or a comment
- [ ] **D4**: `SshConnectionService.ConnectInternalAsync` attaches a copy with the secrets removed to `SshSession` (handling `JumpHost` recursively), confirming no reader relies on `Password`
- [ ] Tests: the dialog does not modify the instance passed in; with "Remember password" unticked, the hand-typed password is **not** persisted through any of the three paths drag to group, duplicate profile and certificate trust (asserted with an in-memory repository spy); `SshSession.ConnectionInfo.Password` is null (v3: only "the dialog conversion returns a copy" has a test case; the spy assertions for the three save paths and D4 are still owed)
- [x] Add a section to `plan.md` describing the fix (together with the #550 section)

### Phase 0b · Foundations (invisible to users)

> v3: shipped with #550; the first registered source is the local shared credentials (§0.1). The items that only concern external sources (per-attempt cache, serialized unlocking, interactive / non-interactive) are left for phase 1a.

- [x] Create `Core/Credentials/`: all contracts of §5.2, `CredentialReference`, `CredentialNotice` (v3: contracts narrowed per §0.1; the notice bar's context is `SharedCredentialPromptContext` in the UI layer, and there is no `CredentialNotice`)
- [x] `SessionProfile.CredentialSource`, added to `Clone()` and `SessionProfileCloneTests.FullyPopulated()`
- [x] Add a case to `ModelSerializationTests`: old data without the field reads as null; new data survives a serialization round trip unchanged (v3: written in `CredentialModelTests`)
- [x] The repository invariant (§5.4), with tests (v3: strips all authentication material)
- [ ] `CredentialResolver`: provider lookup, per-attempt cache, serialized unlocking per provider, queued requests ending immediately after a cancel, the username rule (v3: provider lookup and the username rule are done; the other three come with external sources)
- [x] Wire the five construction points (§5.5), with the precedence rule (v3: FTP and the three plugin points are wired in `MainWindowViewModel`)
- [ ] `ConnectProfileAsync` / `TestConnectionAsync` gain `CredentialResolveOptions`; `ReconnectTabAsync` gains an `interactive` parameter, passed by each caller per §5.7; `TunnelPanelViewModel.AutoReconnectAsync` passes `false` (v3: the local source has no use for it; comes with phase 1a)
- [x] The six decision points (§2.3) recognize references
- [x] Add a `CredentialProviderException` branch to the five retry loops (§5.8); `AuthenticationDialogViewModel` gains the notice and `hideRememberPassword` parameters (notice-bar UI deferred to 1c) (v3: the parameter is one `SharedCredentialPromptContext`, and the notice bar and the "Also update the shared credential" UI are already done along with it)
- [x] `ConnectionDiagnosticsService.HasStoredCredentials` recognizes references
- [x] DI: `ICredentialResolver` registered; with no provider registered, behaviour is exactly the same as today
- [ ] Move `SecureStringConvert` down to Core (§16 N7) (v3: the local source has no use for it; comes with phase 1a)
- [ ] Tests: all invariants in §13.1 (v3: I1, I2, I3, I4, I5, the "hand-typed entry wins" part of I6 and the reference part of I12 have test cases; the per-attempt cache of I6, and I7 and I8, come with external sources; I9–I11 come with the remainder of D1 / D4)

### Phase 1a · Runner and unlock components

- [ ] `CliCredentialRunner` (§7), with a unit test for every hard rule
- [ ] Unlock session cache (§6): TTL timer, zeroing, subscription to `ConnectivityMonitor.Resumed`, "Lock now"
- [ ] Reading and writing the `CredentialProviderSettings` local document (§10.1), clamped in `Normalize`
- [ ] `ICredentialUnlockPrompt` + `CredentialUnlockDialogService` + unlock dialog (§10.6)
- [ ] Audit events (§5.11)
- [ ] Test fixture: the fake CLI (§13.3)

### Phase 1b · The three providers (**finish the "to be tested" items in each part of §8 before starting**)

- [ ] **Test list**, with results written back into §8:
  - 1Password: authorization scope on Windows, whether `account list` triggers authorization, the `purpose` field, whether JSON output needs `--reveal`, error texts
  - Bitwarden: how `bw.cmd` is invoked, the text for a wrong unlock secret, startup time
  - KeePassXC: the output format of `ls -f`, combining `-s` and `-a`, the text for an open failure, KDF time, output encoding on Windows
- [ ] `OnePasswordCliProvider`
- [ ] `BitwardenCliProvider`
- [ ] `KeePassXcCliProvider`
- [ ] Register all three in DI by "register the concrete type first, then forward" (see `:70-81`)
- [ ] Unit tests for each vendor's output parsing (recorded, redacted samples checked in)

### Phase 1c · UI

> v3: the shared-credentials share of the UI already exists (Settings → Shared Credentials page, the credential editor, the "Credential source" dropdown on the connection profile page, the login dialog notice bar, all five resx files; see §0.1).
> The items below refer to the **external-source** share: configuration of the three providers on that page, external items appearing in the dropdown, the item picker and the unlock dialog.

- [ ] Settings page "Credential managers" (§10.2), with the four registration changes and the test updates
- [ ] Source switching, item card and username override on the connection profile page (§10.3)
- [ ] Item picker (§10.4)
- [ ] Login dialog notice bar (§10.5)
- [ ] A new "Credential source" step in connection diagnostics (§5.12)
- [ ] The five resx files
- [ ] Headless UI tests (note: `Dispatch` must use the overload with a return value)
- [ ] Documentation sync (§14)

### Phase 2 · System keychain provider

- [ ] Wait for the `IMasterKeyStore` P/Invoke layer to be in place
- [ ] Windows `CredEnumerateW` / `CredReadW` provider (§9)
- [ ] macOS / Linux settled together with the research document

### Parallel and independent tracks

- **Parallel track** · `IMasterKeyStore`: phases 0/1 of the research document, independent of this document.
- **Independent track** · SSH agent: private-key authentication, waiting for the upstream PR to be merged.

---

## 13. Test plan

### 13.1 Invariants phases 0a / 0b must pin down

| # | Invariant | Test location | Means |
|---|---|---|---|
| I1 | For a profile with a reference, `Password` is empty after persisting through any save path | `VelaShell.Infrastructure.Tests` (repository) | Real SonnetDB (see `SonnetDbPersistenceTests`) |
| I2 | Resolved plaintext never reaches `SaveSessionAsync` | `VelaShell.Presentation.Tests` (workflow) | NSubstitute stand-in repository, asserting the received profile's `Password` is empty |
| I3 | Sync payloads (encrypted and unencrypted) contain no resolved plaintext | `VelaShell.Infrastructure.Tests/Sync` | See `GistSyncServiceTests` |
| I4 | When the resolver throws, what goes out is not the reference string but the dialog; without a dialog delegate a connection error is reported | `VelaShell.Tests` (`MainWindowViewModel`) | Stand-in resolver, asserting `InteractiveAuthenticator` was called and `ConnectProfileAsync` was not called with a wrong password |
| I5 | Each hop of a jump chain is resolved separately; a jump-host resolution failure does not show the login dialog | Presentation.Tests | Stand-in resolver recording calls |
| I6 | Authentication retries within one attempt do not call the provider again; a hand-typed password wins | Presentation.Tests / VelaShell.Tests | — |
| I7 | N concurrent resolutions hitting the same locked provider trigger only one unlock; after the user cancels, the others end immediately | Infrastructure.Tests (`CredentialResolver`) | Stand-in `ICredentialUnlockPrompt` counting calls |
| I8 | Non-interactive mode never calls `ICredentialUnlockPrompt` | Same as above | — |
| I9 | The login dialog does not modify the instance passed in (D3) | VelaShell.Tests (headless) | Reference and field comparison before and after |
| I10 | With "Remember password" unticked, drag to group, duplicate profile and certificate trust do not persist the hand-typed password (D1) | VelaShell.Tests | In-memory repository spy |
| I11 | `SshSession.ConnectionInfo` contains no secrets (D4) | Infrastructure.Tests (`SshConnectionService`, stand-in client factory) | — |
| I12 | Every connection decision point recognizes references | Each in its own project | `DataRow` covering three AuthMethods × with / without reference × override username or not |

### 13.2 CLI runner

One test per hard rule of §7.2:

- the secret passed in never appears in the arguments;
- after a call, neither `BW_SESSION` nor `VELA_BW_PW` exists in the current process's environment;
- an inherited `BW_SESSION` is deleted in the child process;
- stdin is closed after writing;
- a timeout kills the whole process tree (the fake CLI starts another child process; assert both are ended);
- stdout over the limit is truncated and reports `Malformed`;
- e-mail addresses and paths in stderr are replaced;
- non-ASCII characters survive the round trip unchanged.

### 13.3 Fake CLI fixture

- **Create** `tests/fixtures/VelaShell.FakeCredentialCli`: a console program that, following a JSON script pointed to by the environment variable `VELA_FAKE_CLI_SCRIPT`, imitates the output, exit codes, delays, stdin reads, spawning of further child processes and so on of `op` / `bw` / `keepassxc-cli`.
- **How it is referenced**: like the existing fixtures — the test project references it with ProjectReference, the build output lands in the test bin directory, and tests locate it with `typeof(...).Assembly.Location` (`tests/fixtures/README.md`).
- The **flow tests** of the three providers (status probe → unlock → search → resolve → failure mapping) all run on the fake CLI and need no real installation.

### 13.4 Output-parsing samples

- For each vendor, real output recorded during testing is **redacted** and stored in `tests/VelaShell.Infrastructure.Tests/Credentials/Samples/`, with the CLI version in the file name.
- When a CLI upgrade changes the output, add new samples and **do not delete old ones**, to keep compatibility.

### 13.5 Real-CLI integration tests

- Gated by environment variables: `VELASHELL_OP_TESTS=1` / `VELASHELL_BW_TESTS=1` / `VELASHELL_KPXC_TESTS=1`; when unset, `Assert.Inconclusive(...)` (see `CrossPlatformPublishTests.cs:92-105`).
- For KeePassXC a test `.kdbx` can live in the repository (the secret written in the test code, the database contents all fake data), so CI can run it as long as `keepassxc-cli` is installed.
- 1Password and Bitwarden need real accounts and are **run only locally by hand**, with the results recorded in `plan.md`.

### 13.6 UI tests

- Settings page: the new page is automatically included in `SettingsLazyPagesUiTests`, `SettingsBindingErrorTests` and `AccessibleNameTests`.
- Profile page: visibility of the password box, "Remember password" and the item card when switching the source; the confirmation dialog when switching; the output of `BuildProfile`.
- Login dialog: notice bar display and extra action; "Remember password" hidden when there is a reference.
- Unlock dialog: cancel returns null; Esc equals cancel; fail-closed when the main window is missing.
- Headless tests always use `Dispatch(async () => { …; return true; })`.

---

## 14. Documentation and repository sync checklist

| Location | Change | When |
|---|---|---|
| This document | Revise in place when conclusions change during implementation; remove the "to be tested" marks after testing; append revision entries to §18 | Each phase |
| `en/host/` | This document used to be Chinese-only and was listed as such in `en/host/README.md` and the root README; this English translation now exists and is kept in sync with the Chinese original | — |
| [12 Security threat model §1](../plugins/12-security-threat-model.md) | Add: credentials briefly appear in the CLI child process's pipes; unlock sessions are kept in memory for the TTL; passwords received by plugin protocols may come from a vault | Phase 1 |
| [settings-audit.md](settings-audit.md) | Register the fields of `CredentialProviderSettings` and their readers | Phase 1c |
| Host `SECURITY.md` | Add to the scope "credential manager integration: CLI invocation, unlock session cache" | Phase 1 |
| Host `PRIVACY.md` | Explain that VelaShell itself makes no outbound calls, but CLIs such as `bw sync` go online on their own; reference metadata (item-title snapshots, username snapshots) travels with Gist sync | Phase 1 |
| Host README (Chinese and English) | Update the "not available yet" line | Phase 1 |
| Host `plan.md` | Add a section for phase 0a and after each phase completes | Each phase |
| Host `feature-plan.md` | Strike this item when done; if the defects in §3 are not fixed, register them in the backlog | Each phase |
| Host `DESIGN.md` | If §16 N10 is adopted, add style rules for the shared notice bar and segmented control | Phase 1c |
| Keyboard shortcuts | This feature adds no shortcuts; if one is added, change `ShortcutCatalog` first | — |

---

## 15. Risk register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | 1Password on Windows needs Windows Hello for every call to `op` | Medium | High: a very poor experience | First test of phase 1b; one call to `op` per connection; picker results cached |
| R2 | CLI output format drifts between versions | High | Medium | `JsonDocument` picking fields as needed; sample tests; the settings page shows the version; `Malformed` names the step that failed |
| R3 | Passwords returned by `bw list items` linger in memory or reach the log | Medium | High | Parse as a stream and discard immediately; stdout never logged; dedicated tests |
| R4 | Sensitive environment variables leak into the local terminal | Low (there are rules) | High | §7.2 rules 4 and 5; tests; pinned down by code comments |
| R5 | KeePassXC addresses by path and picks the wrong entry when names collide | Medium | High: the wrong password for the wrong machine, or rejection by the server | Duplicate check and warning at selection; `NotFound` / rejection leads to choosing again |
| R6 | KeePassXC re-derives the key on every call, making it slow | Medium | Medium | Merge calls; if needed, the resident-process approach of §16 N8 |
| R7 | A broken reference fails silently | High | Low | `NotFound` notice bar + "Choose the item again"; no background checks |
| R8 | After cross-device sync, the other machine lacks the provider | High | Low | `NotInstalled` falls back to typing in the dialog; the connection is not blocked |
| R9 | The unlock dialog is used as a phishing vehicle, or makes users suspicious | Low | Medium | Appears only within an operation chain the user started; shows the provider, target and reason; automatic reconnect never prompts |
| R10 | A storm of automatic reconnects triggers lots of CLI calls | Medium | Medium | Non-interactive mode fails fast when locked; serialized per provider |
| R11 | The phase 0a fixes change users' habits (for example whether retrying from the failure card needs the password again) | Low | Low | The closure captures the latest `current`, keeping the current behaviour |
| R12 | `bw lock` breaks the user's own CLI session | — | Medium | Explicitly never called (§8.2) |
| R13 | In-process plugins receive vault passwords | — | Medium | The same trust level as hand-typed passwords today; stated in the threat model |

---

## 16. New decisions awaiting confirmation

These are new questions that came up while combing through v2. Each has a suggestion; **please confirm them one by one**.

| # | Question | Options | Suggestion | Reason |
|---|---|---|---|---|
| N1 | How to handle the existing defects of §3 | A. Fix first as phase 0a / B. Fold into phase 0b / C. Only register them, no fix | **A** | D1 goes against the user's explicit choice, has nothing to do with this feature and deserves its own fix; fixing it first lets 0b stand on a clean foundation |
| N2 | Is the private-key passphrase governed by "Remember password" (D2) | A. Governed; add the stripping / B. Not governed; just add a comment | **B**, plus a line of explanation next to the passphrase box on the profile page | The profile page's "Remember password" appears only for password authentication, and the passphrase has always been saved with the profile; making it governed would make existing users' passphrases disappear after their next connection |
| N3 | Automatic reconnect hits a locked vault | A. Non-interactive, fail fast / B. Show the unlock dialog as usual | **A** | §5.7; avoids prompts while unattended |
| N4 | Where provider settings live | A. A separate local document / B. `AppSettings` | **A** | §10.1; local paths stay out of sync and export by construction |
| N5 | Settings entry point | A. A new separate settings page / B. Inside "General → Security and Privacy" | **A** | A lot of content; the change points and tests for a new page are well defined |
| N6 | Clearing the unlock cache on sleep or screen lock | A. Reuse `ConnectivityMonitor.Resumed` / B. Bring in `SystemEvents`, overturning the earlier trade-off / C. Rely on the TTL alone | **A** | Zero new dependencies, and the wake case is covered; 1Password itself revokes authorization when the app locks |
| N7 | `SecureStringConvert` | A. Move down to Core / B. Write a second copy in Infrastructure | **A** | It depends only on the BCL and has nothing to do with the UI; two implementations will drift apart sooner or later |
| N8 | How KeePassXC is called | A. v1 calls per operation with merged arguments, optimise if slow / B. Build the resident `open` process from the start | **A** | A resident process keeps the database unlocked, and its output is harder to parse; let measurements decide first |
| N9 | Audit granularity | A. Only failures and unlock events / B. Every resolution | **A** | Every connection already has a `connect` audit entry; more would flood it |
| N10 | Extract shared styles along the way (notice bar, segmented control) | A. Do it with this feature and update DESIGN.md / B. Don't; copy locally | **B**, as a separate backlog item | Avoids widening this feature's scope; with five or six notice bars each written their own way, it deserves its own clean-up |
| N11 | If 1Password's status probe triggers authorization | A. Probe only the version and show the sign-in state as "Unknown" / B. Probe anyway and accept the prompt | **A** | Windows Hello popping up as soon as the settings page opens would confuse users |

---

## 17. Small questions to settle during implementation

These do not block the start of phase 0; they are decided at the phase 1 review:

1. Is the 15-minute default for `UnlockCacheMinutes` right (maximum tentatively 480)?
2. Should the passphrase of a local private-key file also be fetchable from the vault (§11)?
3. 1Password with multiple accounts: is the account chosen as a global default on the settings page, or per item in the picker (tentative in this document: a default in settings, switchable in the picker)?
4. When a KeePassXC database alias is renamed, should the profiles referencing it be updated in bulk automatically?
5. The retry limit for a wrong Bitwarden unlock secret (tentatively 3).
6. Should the item picker remember the last provider and database chosen (stored in the local settings document)?

---

## 18. Revision history

| Date | Version | Contents |
|---|---|---|
| 2026-09-15 | v1 | First draft: correcting the premise, architecture, phasing; 8 decisions finalised |
| 2026-09-15 | v2 | Verified file by file the connection path, settings and UI, and wiring and tests; added the three carriers and five construction points, the full table of connection entry points, the full table of persistence points and the in-memory residence table; found four existing defects (§3); checked the three CLIs against official documentation and source code (§8), correcting unverified statements such as "Bitwarden takes about 1 second" and "KeePassXC can address by UUID"; added interactive and non-interactive resolution, the repository invariant, the CLI runner specification, the fake CLI fixture, the per-phase task list and the risk register; added 11 decisions awaiting confirmation (§16) |
| 2026-10-03 | v3 | Host #550 shipped the first source, the built-in "shared credentials" (provider id `shared`), together with D1 / D3 of phase 0a and most of the foundations of phase 0b; added §0.1 recording each difference from v2 (`ResolvedCredential` carries the authentication method and private key, the username becomes "the connection's wins if filled in", the reference keeps only two fields, the invariant strips all authentication material, resolution returns a profile copy, jump-host failures do not fall back to the login dialog, the plugin construction points are wired in `MainWindowViewModel`, sync payload version 3); the decision table in §0 and the task list in §12 annotated accordingly |
