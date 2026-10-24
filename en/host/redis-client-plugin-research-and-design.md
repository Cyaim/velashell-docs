# A Full GUI Redis Client: Plugin Research and Design

> Written: 2026-08-17　Baseline code: `main` @ `6daaf1a`
>
> Prerequisite reading: [`../templates/dev-guide.md`](../templates/dev-guide.md) (capability surface and rules),
> [`s3-protocol-plugin-design.md`](s3-protocol-plugin-design.md) (the trade-offs and lessons of the first protocol plugin),
> [`VelaShell/DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md) (design tokens and component rules).
>
> This document answers three questions: **can a full GUI Redis client be built as a plugin**;
> **how many openings the host has to provide for it**; and **what the UI has to look like to deserve the phrase "keyboard first, high information density"**.
> Section 6 (UI design) is the core; every other section serves it.

---

## 1. Conclusion first

**It can be done, and it should be a plugin.** But **it cannot copy the S3 route** — the S3 plugin goes through the "protocol capability domain"
(`IProtocolFileSystem`), whose premise is "this protocol looks like a file system". Redis does not: it has no directories,
no files; the keyspace is a flat mapping from byte strings to several data structures, values come in six or more native shapes,
and what operators actually need to see (TTL, encoding, memory usage, slow log, connection count, replication state) has nowhere to go in a file model.
Forcing it into the dual-pane file browser would only produce a fake client that "can look but can't be used".

So the conclusion has three parts:

| # | Conclusion |
|---|---|
| 1 | **The host opens a new contribution point: the "workspace connection type"** (`contributes.workspaces`). It shares the **entire existing machinery** of protocol contributions — the connection dialog, declarative forms, credential encryption, lazy activation — and the only difference is that when a session is opened the host asks the plugin for a `Control` and hosts it as a dock document, instead of opening the dual-pane file browser. The change surface is small (see §3), and in return Redis connections become **first-class citizens just like SSH/SFTP**: they go into the session tree, recent connections and the command palette, ride Gist cloud sync, and have their passwords AES-encrypted on disk. |
| 2 | **The host also opens a "declarative SSH tunnel"**: pick a saved SSH profile in the connection form, and the host establishes the SSH session + local port forward and hands the plugin the **rewritten local endpoint**. The plugin therefore writes not one line of SSH code and never sees a single credential — and the most frequent scenario, "an operator quickly checks a production Redis", goes from "open a tunnel by hand, then type 127.0.0.1" to "pick a jump host". Once this opening exists, every later MySQL / PostgreSQL / MongoDB / Kafka plugin gets it for free. |
| 3 | **The client library is StackExchange.Redis.** During research this document leaned towards writing a RESP implementation directly (reasons in §4); **the user decided to use the library instead**: speed of delivery comes before completeness at those edges. It has shipped and been verified; the two hard limits the library brings (`MONITOR`, blocking commands) and the pitfalls actually hit are recorded in §4.1 and must not be mistaken for oversights. |

**The right definition of "full" for Redis**: S3 has 116 operations, and any operation the UI does not cover is **unreachable for the user**,
so that design had to count coverage item by item. Redis has about 240 top-level commands (about 460 including subcommands), but once the client ships
a **real CLI console**, **every command is reachable by construction** — the UI's job is no longer "draw all 240 of them"
(that would turn the redis-cli man page into buttons, which is worthless) but to make **the 80 most frequent ones zero-typing**
and keep the 20 dangerous ones from being fired by mistake. This is the fundamental difference between this design and a "feature-checklist" design, and the starting point of every decision in section 6.

**Effort**: about 3–5 days on the host side (two contribution points + the form field shape + tests); about 10–11k lines on the plugin side,
in four milestones (the same order of magnitude as the AI assistant plugin at 11.3k lines and the S3 plugin at 7.4k lines). Item by item in §11.

---

## 2. Why a plugin, and why not a "protocol plugin"

### 2.1 Why a plugin

The same accounting as S3, but an even better deal:

- **Zero cost for users who don't use Redis**: the Redis panel, the RESP engine, the command metadata and about 250 UI strings all live in the plugin folder;
  nothing is loaded until a Redis tab is opened (`onWorkspace:` lazy activation).
- **Decoupled iteration**: the Redis ecosystem keeps moving (hash-field expiration in 7.4, JSON/Search/TimeSeries/Bloom modules bundled from 8.0 on,
  the Valkey fork). None of that should force a host release.
- **It is the best candidate for the "second business plugin"**: `docs/plugins/15-ecosystem-ideas.md` §1 lists "database client"
  in the first tier and marks its complexity "high". It feeds back into the framework's gaps better than a container-management plugin —
  container management only uses `RemoteExec` (an existing capability), while Redis forces out, in one go, the two extension points that are genuinely missing:
  a "non-file connection type" and "tunnels". Those extension points are the shared prerequisite of **every** later database/middleware plugin.

### 2.2 Why the protocol capability domain won't do

Putting Redis into `IProtocolFileSystem` takes a string of lies:

| What the file model requires | The truth about Redis | Consequence of forcing the mapping |
|---|---|---|
| A directory tree | The `:` in key names is just an ordinary character (isomorphic to S3's `/`; this part actually fits) | Tolerable |
| One entry = one byte stream | A value may be a hash / list / set / zset / stream / JSON — **not a byte stream** | What do you get when you double-click a hash to "download" it? |
| Size, modification time, permissions | No mtime, no permission bits; what it has is **TTL, encoding, memory usage** (which the file model has no place for) | The column names lie (§3.1 of the S3 document makes a point of "not filling the group column with the storage class") |
| Read/write = upload/download | A write is `HSET field value`, `ZADD score member`, `XADD` … **with structural semantics** | A transfer queue makes no sense at all |
| Listing a directory = one call | `SCAN` is **cursor-based, not guaranteed complete, and may spin for several empty rounds** | No progress bar can be drawn; "the directory is fully listed" is a lie |

And it gets harder further down: half the value of a Redis client is in **server state** (INFO / SLOWLOG / CLIENT LIST /
memory analysis / Pub-Sub / MONITOR), none of which has anywhere to land in a file browser.

**So what is needed is not "one more protocol" but "a session document rendered entirely by the plugin".**

---

## 3. Three openings the host has to provide

All three are **purely additive** (apiLevel stays 1, compatible with the "within a generation, only add, never change" rule), and they deliberately reuse existing machinery.

### 3.1 Opening 1: the workspace connection type (`contributes.workspaces`)

The manifest declaration (it reaches the connection dialog at discovery time **without touching the assembly**, the same path as protocol contributions):

```jsonc
{
  "id": "velashell.redis",
  "contributes": {
    "workspaces": [
      { "id": "velashell.redis", "displayName": "Redis", "defaultPort": 6379 }
    ]
  },
  "activationEvents": [ "onWorkspace:velashell.redis" ]
  // hostMode must be inProcess: the host asks the plugin for an Avalonia Control to host in the dock area,
  // and native controls cannot be embedded across processes (blueprint 08 has dropped HWND adoption). Validation rejects isolated outright,
  // with the same wording and the same reason as protocol contributions.
}
```

On the SDK side (three new types + one capability domain, **all reusing the field declarations and exception family that already exist on the protocol side**):

```csharp
namespace VelaShell.PluginSdk.Workspaces;

/// <summary>A session document type rendered entirely by the plugin (Redis, MySQL, Kafka…).</summary>
public sealed record WorkspaceDescriptor
{
    public required string Id { get; init; }              // = the plugin id or prefixed by it, all lowercase, immutable once published
    public required string DisplayName { get; init; }
    public int DefaultPort { get; init; }
    public string? HostLabel { get; init; }
    public string? UsernameLabel { get; init; }           // Redis: "ACL username (empty = default)"
    public string? PasswordLabel { get; init; }
    public IReadOnlyList<ProtocolSettingField> Fields { get; init; } = [];   // ← reuses the protocol side's declarative form
    public WorkspaceFeatures Features { get; init; }      // AnonymousAccess | CertificateTrust | SshTunnel
    public string? TrustedThumbprintSettingKey { get; init; }
}

/// <summary>The request when the host opens a workspace session. Fields mirror ProtocolConnectRequest (credentials are single-use).</summary>
public sealed record WorkspaceConnectRequest
{
    public required string SessionId { get; init; }
    public required string Host { get; init; }            // when tunnelled, this is already the local forwarding endpoint
    public required int Port { get; init; }
    public string Username { get; init; } = "";
    public string Password { get; init; } = "";
    public IReadOnlyDictionary<string, string> Settings { get; init; } = ...;
    public string DisplayName { get; init; } = "";
    public WorkspaceTunnelInfo? Tunnel { get; init; }     // non-null means the host has built the tunnel on the plugin's behalf (see 3.2)
}

/// <summary>An open workspace document: the plugin owns the connection and the UI; the host only handles the tab and status display.</summary>
public interface IWorkspaceDocument : IAsyncDisposable
{
    object CreateView();                                  // called on the UI thread, returns an Avalonia Control
    ProtocolSessionState State { get; }                   // ← reuses the protocol side's state enum
    event EventHandler<WorkspaceStatus>? StatusChanged;   // state + one line of status-bar text + optional latency (ms)
    Task ReconnectAsync(CancellationToken ct);            // the host's "Reconnect" button
}

public interface IWorkspaceProvider
{
    Task<IWorkspaceDocument> OpenAsync(WorkspaceConnectRequest request, CancellationToken ct);
}

public interface IWorkspacesApi
{
    IDisposable Register(WorkspaceDescriptor descriptor, IWorkspaceProvider provider);
}
```

Host-side changes (place by place, countable):

| Location | Change |
|---|---|
| `PluginManifestReader` | Recognise `contributes.workspaces` and the `onWorkspace:` event; reject the `isolated` + workspaces combination |
| `PluginProtocolRegistry` | Generalised into a "connection type registry": entries carry a `Kind` (`FileSystem` / `Workspace`). The two-phase declare → register → lazily activate → unregister flow is **reused as is** |
| `ConnectionProfileViewModel` / `ConnectionProfileView` | The tabs come from the merged registry; the form renderer has zero changes (the same `ProtocolSettingField` set) |
| `MainWindowViewModel.OpenPluginDocumentForProfileAsync` | Branch on `Kind`: `FileSystem` → the existing `SftpDocument`; `Workspace` → the new `PluginWorkspaceDocument` (an `IDockViewProvider` wrapping the plugin's `CreateView()`, title = session name, status dot bound to `StatusChanged`) |
| `SessionTreeNodeViewModel` | Icon/double-click semantics taken by `Kind` (`IsDeclared` is a synchronous query, so this does not load the plugin) |
| `ConnectionType` | **Unchanged**. Still `Plugin = 4`, with `PluginProtocolId` remaining the discriminator — "file protocol or workspace" is **the plugin's declaration** and does not belong in the user's configuration; the user's profile only records "I use `velashell.redis`". After the plugin is uninstalled, both kinds degrade to the same sentence, "the plugin that provides this connection type is unavailable", with no need to tell them apart |

> **Why not add a `ConnectionType.PluginWorkspace = 5`**: it would split one thing (a plugin-provided connection)
> into two enum values, even though both have exactly the same storage shape, credential handling and lazy activation; worse, it would freeze `Kind`, a
> **plugin-side property**, into the user's persisted configuration — the day a plugin changes a connection from file type to workspace type (a perfectly legitimate evolution),
> old profiles would be read as something else. This is the same lesson as retired value 3.

### 3.2 Opening 2: the declarative SSH tunnel (`WorkspaceFeatures.SshTunnel`)

**This is the best value for money in the whole design.** A production Redis is almost never exposed to the internet; the real path for operators is
"SSH to the jump host → then connect to Redis". Today the user has to: open a tunnel by hand → remember the local port → enter
`127.0.0.1:<that port>` in the Redis connection → have the Redis connection mysteriously break when the tunnel is closed. All four steps are manual state synchronisation.

Once declared, the host automatically appends a section to the connection form (**the plugin writes no UI code at all**):

```
┌ Via SSH tunnel (optional) ──────────────────────────┐
│ Jump session   [ bastion-01 (10.0.0.9)          ▾ ] │  ← dropdown of saved SSH profiles
│ Target         [ 10.0.3.12          ] : [ 6379 ]    │  ← empty = use the host/port above
└─────────────────────────────────────────────────────┘
```

What the host does when the session is opened: resolve the jump profile → reuse or establish the SSH session (**through the host's existing two-step authentication,
fingerprint verification and ProxyJump chain**) → request a temporary local port from `ITunnelService.CreateLocalForwardAsync` →
rewrite `WorkspaceConnectRequest.Host/Port` to `127.0.0.1:<temporary port>` and attach the real target in `Tunnel`
(only so the UI can show "↝ bastion-01") → tear the tunnel down when the document closes or SSH disconnects.

The only SDK addition needed is one field shape:

```csharp
public enum ProtocolSettingKind { Text, Password, Boolean, Integer, Choice, /* new */ SshSession }
```

A field of the `SshSession` shape persists a **session profile id**, and the host renders it as a "saved SSH profiles" dropdown.
It is just as useful to protocol plugins (S3/WebDAV), and is purely additive.

> **Why not open `ITunnelService` to plugins directly**: then the plugin would have to handle the whole set of "which SSH profile, what if it isn't connected,
> whether to show two-step authentication, what if the fingerprint changed" — all of which are the host's core assets, and credentials are explicitly
> **never to leave the host**. Letting the plugin accept only "a local endpoint that already connects" gives the smallest trust surface and the least code,
> and no future database plugin has to rewrite it. **Get the direction right, and the effort comes out right.**

### 3.3 Opening 3: two small mount points (optional, not blocking v1)

- **Status bar contribution**: put "ops/sec · memory · hit rate" into the host's status bar (an existing gap, already listed in blueprint 08).
  v1 makes do with a status strip drawn inside the document; visually there is little difference.
- **`onSessionConnect` activation event**: let the plugin be woken when the user connects to a machine (for the "probe the remote Redis" of §7.1).
  v1 can use a command-palette command instead (triggered by the user), which is only one step worse.

---

## 4. The client engine: write RESP directly, or StackExchange.Redis

The lesson the S3 document left behind is that "**a dependency trade-off must first measure the protocol surface, and only then talk about two hundred lines vs a package**".
Apply the same yardstick to Redis and the conclusion is **the opposite of S3's** — because what gets measured is the **shape** of the protocol surface, not the number of operations:

| Dimension | S3 (116 operations) | Redis (about 240 commands) |
|---|---|---|
| Encoding/decoding per operation | **All different**: every operation has its own XML request/response + dedicated error codes | **Completely uniform**: everything goes in as an array of bulk strings and comes out as one of five (RESP2) / fourteen (RESP3) types |
| Marginal cost of adding an operation | One serialiser + one error mapping | **≈ 0** (one more command name) |
| How much the SDK writes for you | Thousands of lines of plumbing | Almost nothing; what it writes for you is **connection management**, not encoding |
| What the SDK blocks | Nothing | See below |

What StackExchange.Redis (MIT, licence-compatible) really provides is a **multiplexed connection pool** — which is exactly
what a GUI client **needs least, and what gets in its way**:

1. **Blocking commands are explicitly unsupported.** The official documentation says it plainly: under multiplexing, `BLPOP`/`BRPOP`/`BRPOPLPUSH`
   would stall the whole multiplexer, so the library does not provide them. But in a client that calls itself complete, a user typing `BLPOP q 5`
   in the console must get it to run.
2. **`MONITOR` has no corresponding API.** It requires the connection to enter a "stream only, never reply" state, which inherently conflicts with the multiplexing model.
   And `MONITOR` is a hard requirement for troubleshooting.
3. **The console needs "sticky connection state".** `SELECT` / `MULTI`-`EXEC` / `WATCH` / `SUBSCRIBE` / `CLIENT NO-TOUCH`
   are all **connection-level** state. Multiplexing dispatches the user's commands to any physical connection — the user types `MULTI` and then `EXEC`,
   and the two may not land on the same socket. That is not "inconvenient", it is **wrong**.
4. **RESP fidelity.** The console has to present reply types as faithfully as redis-cli does (`(integer) 3` / `(nil)` /
   `(empty array)` / the nested indentation of `1) 1) "a"` / RESP3 maps and sets). Going through the
   `RedisResult` abstraction and then reconstructing means removing information and guessing it back.
5. **It says extra things on your behalf.** Things like `CONFIG GET` / `CLIENT SETINFO` during the connection handshake turn into
   "cannot connect" or a string of warnings on managed Redis instances that disable `CONFIG`/`CLIENT` — and those instances are exactly what operators connect to most.

**The actual size of writing it directly** (one-off, with near-zero maintenance afterwards):

| Component | Lines | Notes |
|---|---|---|
| RESP2/RESP3 reader/writer | ~450 | Built on `System.IO.Pipelines`; 14 types + attribute/push frames |
| Connection (TCP/TLS/AUTH/HELLO, exclusive command turns, pipelining) | ~500 | One `SemaphoreSlim(1)` per socket, no multiplexing |
| Topology (CRC16 slots, `MOVED`/`ASK`, `CLUSTER SHARDS`, Sentinel discovery and `+switch-master`) | ~550 | The CRC16 table is 30 lines; cluster routing is pure arithmetic |
| Command metadata (`COMMAND DOCS`/`COMMAND INFO` cache + fallback table for older versions) | ~250 | See §6.4 — this data **comes from the server**, which also removes the worry that writing it directly means "maintaining the command table yourself" |
| Total | **~1750** | Zero third-party dependencies; the plugin package grows by not a single byte |

**Trade-offs and costs** (written down so they are not taken for oversights later): reconnect backoff, cluster topology refresh and Sentinel failover —
these "edges the library has thought about for you" have to be thought through yourself, and **pinned down with a loopback server** (§10).
But a GUI client's concurrency model is far simpler than a server application's — **there is no contention**: a panel has a single-digit number of commands
in flight at once, and the user is waiting for the results. That is exactly the kind of scenario where "the risk surface of writing it yourself" is smallest.

> Taken together with the judgement in §2: S3 chose the SDK and Redis (per the research conclusion) leaned towards writing it directly, and both conclusions use the same rule
> — **the shape of the protocol surface decides the dependency trade-off**, not "write it yourself whenever you can" or "use a library whenever there is one".

### 4.1 The actual decision: use StackExchange.Redis (2026-08-17, shipped)

**The user decided to switch to the library**, on the grounds that "writing it directly takes too long; with the library we can deliver results straight away". This is an explicit
trade of completeness for speed, and the call is the user's to make. The measured results after shipping (`StackExchange.Redis 3.1.13`, MIT):

| Item | Actual result |
|---|---|
| Delivery speed | Connecting / probing / cursor scanning / paged value fetching all work; about 2.4k lines on the plugin side (the 1.75k-line estimate for writing it directly covered only the encoding layer, not the tests it would need) |
| Package size | 2.5 MB plugin folder: `StackExchange.Redis.dll` 2.0 MB + `RESPite.dll` 110 KB + `System.IO.Hashing.dll` 55 KB + the plugin itself 165 KB. Distributed on demand; users who don't use Redis don't pay for it |
| `MONITOR` | **Confirmed unavailable**: the library does not provide it under the multiplexing model. The console will refuse it truthfully and explain why, rather than pretend it can run |
| Blocking commands (`BLPOP` etc.) | **Confirmed unavailable** (stated in the official documentation). Same as above: refused truthfully |
| Sticky connection state in the console (`MULTI`/`WATCH`/`SELECT`) | Still unsolved: multiplexing dispatches commands to any physical connection. This must be faced head-on when the console ships; the options are below |
| `INFO`/`CONFIG`/`DBSIZE` | Need `AllowAdmin = true`. Dangerous commands are still stopped by the plugin's own guardrails (based on the `COMMAND INFO` flags), not by this master switch in the library |
| RESP3 | `Protocol = RedisProtocol.Resp3`, falling back automatically on older versions; the protocol actually negotiated is probed with a raw `HELLO` and recorded on the status strip |

**Three options for when the console ships** (pick one then; do not fudge it):
① use the library's `Execute` only for stateless commands, and open a separate raw socket for `MULTI`/`SUBSCRIBE`/`MONITOR`;
② send everything through a raw socket (which amounts to adding the direct RESP implementation back, but only for the console, a much smaller amount);
③ state truthfully that these kinds of commands are unsupported. **Leaning towards ①** — it keeps the library's value on the data plane and
confines what the library blocks to one separate connection.

### 4.2 Pitfalls actually hit (a real Redis 3.0.504)

This machine happens to run Redis 3.0, so every degradation path got exercised, with two real gains:

1. **`SCAN … TYPE` only exists from 6.0; older servers reply `ERR syntax error`.** The first implementation "fell back to a scan without
   TYPE" and handed the results over as they were — so the UI would **say "type: hash" while listing keys of every type**.
   That is the worst kind of bad: the UI is lying and the user has no clue. The fix is capability probing (one syntax error is a
   definite answer) + a client-side batched `TYPE` to apply the filter: the cost is one extra round trip per page on old servers,
   in exchange for "ask for hashes, get only hashes" holding on every version. Pinned by `ScanAsync_WithTypeFilter_OnlyReturnsThatType`
   — and that test only really goes down the client-side narrowing branch on 3.0.
2. **`MEMORY USAGE` only exists from 4.0.** When it cannot be obtained, the result must be `-1` (unknown) and **never 0** —
   0 would be read by the UI as "uses no memory". This is the numeric form of the §6.8 rule "an empty state is not an error".

> The lesson shares its root with the S3 document: **a degradation path only counts as verified on a machine that actually degraded.** One old server
> is more useful than ten comments saying "this will fall back in theory".

---

## 5. Connections, topology and the "division of connections"

### 5.1 The connection form (declarative, rendered by the host)

| Field | Shape | Default | Advanced | Notes |
|---|---|---|---|---|
| Host / port | Built into the host | `6379` | | `HostLabel = "Server address"` |
| Username | Built into the host | empty | | `UsernameLabel = "ACL user (empty = default)"` |
| Password | Built into the host | empty | | AES-encrypted on disk, through the existing "Remember password" switch |
| Deployment mode | Choice | `Standalone` | | `Standalone` / `Sentinel` / `Cluster` |
| Master name | Text | — | | **Shown only in Sentinel mode** (`VisibleWhen = ("mode","sentinel")`); the argument to `SENTINEL get-master-addr-by-name` |
| Default database | Integer | `0` | | **Not shown in Cluster mode** (a cluster only has db0, so a value would mean nothing) |
| Use TLS | Boolean | `false` | | Self-signed certificates go through "prompt → remember the fingerprint → reconnect" (the same flow and wording as FTPS/S3) |
| Environment tag | Choice | `Development` | | `Production` / `Staging` / `Development` — drives the strength of the §6.7 guardrails and the tab colour |
| Read-only mode | Boolean | per environment | | On by default for production; see §6.7 |
| Key separator | Text | `:` | ✓ | The level separator of the key tree |
| SCAN batch size | Integer | `500` | ✓ | `SCAN COUNT` |
| Group fold threshold | Integer | `8` | ✓ | How many keys with the same prefix it takes before they fold into one row; `1` = never fold (see §6.2.1) |
| Per-batch limit | Integer | `5000` | ✓ | Soft cap for automatic scanning; scanning stops there and suggests narrowing |
| Value preview limit | Integer | `262144` | ✓ | Beyond it only the first N bytes are fetched, and the UI says plainly that the value is truncated |
| Client name | Text | `velashell` | ✓ | `CLIENT SETNAME`, so a DBA can recognise who it is in `CLIENT LIST` |
| TLS fingerprint | Hidden | — | | The write-back slot for `TrustedThumbprintSettingKey` |

Ten of the fifteen fields go on the "Advanced" page — a rule learned from S3 plugin users' feedback (the connection dialog used to be one long column,
and with enough fields it ran off the screen so the buttons at the bottom were out of reach); the ones that decide "does it connect at all" stay on the "General" page.

The two fields that only make sense in one mode are **declared** away with `VisibleWhen`, instead of leaving a box with a line of small print explaining it.
Both used to be "shown + small print": under Master name it said "Sentinel mode only", under Default database "ignored in Cluster mode" —
using copy to explain a field that should have disappeared pushes the UI's job onto the copy. When the condition does not hold the field disappears,
but **a stored value is still kept and passed back**: someone who filled in a master name under Sentinel, switched to Standalone for a look and then back,
should not find that the UI has wiped what they typed.

The "Test" button takes the plugin's own path for plugin connections (really opening a session and closing it again), not SSH —
see the third row of §11.3.

### 5.2 Division of connections: at most four sockets per session

Multiplexing is wrong (§4), but "a new connection per operation" is even more wrong. The split follows **connection-level state**:

| Connection | When it is established | What it carries | Why it can't be merged |
|---|---|---|---|
| **Control** | As soon as the document opens | Key scanning, value fetching, writes, INFO polling | — |
| **Console** | When the console is first expanded | Everything the user types by hand | The user's `SELECT 5` / `MULTI` / `BLPOP` must not pollute the browser; conversely, the browser's background polling must not slip into the user's transaction |
| **Subscription** | On the first subscription | `SUBSCRIBE`/`PSUBSCRIBE`, keyspace notifications | Under RESP2 a connection in subscribed state only accepts the subscribe family of commands (RESP3 could share one, but keeping them separate is still more predictable) |
| **MONITOR** | When the user explicitly turns it on | The `MONITOR` stream only | Once in, the only ways out are `RESET` (6.2+) or disconnecting |

In Cluster mode the "control" connection is established lazily per node (one per node), and `SCAN`/`DBSIZE`/`INFO` run node by node and are then aggregated.
**Idle subscription/MONITOR connections close automatically once their panel is collapsed** — the host is a terminal application that is sensitive to memory and file descriptors.

### 5.3 Capability probing beats version checks

`redis_version` in `INFO server` cannot be trusted on managed instances and forks (Valkey / KeyDB / Dragonfly /
Upstash / every cloud have their own version numbering, and often disable or rename commands). So every optional capability is **probed**:

```
COMMAND INFO scan client object memory      → which commands exist, with their flags and key specs
COMMAND DOCS  (7.0+)                        → the data source for console completion and argument hints
CONFIG GET databases                        → if unavailable, draw 16 databases and mark it "not confirmed"
MODULE LIST / COMMAND INFO json.get         → whether the modules panel appears
```

This shares its root with pitfall 4 in the S3 document: **"never configured" and "not supported" are not errors, they are empty states**.
It is even more common on Redis — managed instances routinely disable `CONFIG`/`CLIENT`/`DEBUG`/`CLUSTER`,
and treating that as an error would make the overview page a sea of red the moment it opens.

---

## 6. UI design (the core)

### 6.1 Information architecture: one dock document + one bottom drawer

The four kinds of work in a Redis client run **in parallel**, not in sequence: browsing keys, editing values, typing commands, watching server state.
Making them mutually exclusive top-level tabs (RedisInsight's Browser / Workbench / Analysis are mutually exclusive) means that
for the most common action, "I changed a value and want to check it", the user has to switch pages back and forth, and loses context whenever they switch away.

This design takes "**a left/right split body + a bottom drawer**" — exactly the shape this application's users know best (terminal below, content above),
and naturally compatible with VelaDock: when it isn't enough, drag the whole document into a split and put it next to a terminal.

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│ ● prod-cache  10.0.3.12:6379 ↝ bastion-01  [db0 (1.2M) ▾]  [Read-only]  [Production]  ⟳   ✕   │ 36  VelaBgSidebar
├─────────────────────────────────┬─────────────────────────────────────────────────────────────┤
│ ⌕ user:                         │ user:10086:profile          [hash]  TTL 29:58    1.4 KB     │ 36
│  ⟨Prefix⟩ Contains Glob [Type▾] │ listpack · 12 fields             [Edit TTL][Rename][Delete] │
│  → SCAN MATCH user:* …          │─────────────────────────────────────────────────────────────│ 26  VelaBgSurface
├─────────────────────────────────┤ Field             Value                                     │
│ Name                          # │─────────────────────────────────────────────────────────────│
│ ▾ user                     (842)│ name              张三                                   ✎  │ 28
│   ▾ 10086                    (3)│ age               32                                     ✎  │
│       profile              hash │ tags              ["vip","beta"]                         ✎  │
│       sessions             zset │ …                                                           │
│       lock               string │ + Add field                     HSCAN 12/12, all read       │ 28
│ ▸ order                   (1.2k)│                                                             │
│ ▸ session                 (3.1k)│                                                             │
├─────────────────────────────────┴─────────────────────────────────────────────────────────────┤
│ Scanned 5,000 / ~1,240,000 (0.4%)   ⟨Continue scan⟩ ⟨Stop⟩          RESP3 · 7.2.4 · 12 ms     │ 24  VelaBgSidebar
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│ Console │ Overview │ Slow log │ Clients │ Subscriptions │ Memory analysis        ⌃ Collapse   │ 28
│ 10.0.3.12:6379[db0]> hgetall user:10086:profile                                               │
│ 1) "name"                                                                                     │
│ 2) "\xe5\xbc\xa0\xe4\xb8\x89"   张三                                                          │
│ 10.0.3.12:6379[db0]> ▊                                                                        │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

All geometry and colour come from `DESIGN.md`, with no new tokens invented: 36px header / 26px column header / 28px rows / 24px status strip,
the three layers `VelaBgSidebar`→`VelaBgSurface`→`VelaBgTerminal`, key names and values always in `VelaUiMonoFont` 11px,
type badges with a `VelaAccentDim` background + `VelaAccent` text (`CornerRadius:2`, the same spec as StatusTag).
The environment tag is the **only** place allowed to change the primary colour: production `VelaError`, staging `VelaWarning`, development `VelaTextTertiary`
— it appears as a suffix on the tab text and in the document header, so that "I am touching production right now" cannot be overlooked.

### 6.2 The keyspace browser: a flat list + three rules

**Rule 1: never `KEYS`.** Always `SCAN`. This is not a performance preference — `KEYS *` on millions of keys **blocks the whole instance**,
and a GUI client doing that is handing the user a gun with no safety. The price is that "total key count" and "scan completeness" become estimates,
hence:

**Rule 2: progress must be honest.** The status strip always states three things: how many keys have been scanned, the estimated total from `DBSIZE`,
and whether the cursor has returned to 0. **Only when the cursor is back to zero may it say "fully scanned"**; every other wording is "scanned so far".
The same goes for the counts on group rows — the 40 in `demo:order:2026:*  40` is the number **scanned so far**, and hovering gives the full explanation.
(This is the same rule as the S3 document's refusal to "fill the group column with the storage class": **the UI may not lie**, even when the truth is wordier.)

**Rule 3: one round trip per batch.** Each round: `SCAN <cur> MATCH <p> COUNT 500 [TYPE t]` fetches a page of keys,
then `TYPE`/`TTL` for that page are sent as **one pipelined** round trip (with `SCAN ... TYPE` on Redis 6.0+,
even `TYPE` is skipped). Memory usage (`MEMORY USAGE`) is **not fetched while scanning** —
it is an optional column, and turning it on says explicitly "one more command per key".

**The filter bar's semantics must be visible.** Three modes, ⟨Prefix⟩/Contains/Glob, + a type dropdown, with **the command actually being sent echoed live** underneath:

```
⌕ user:            ⟨Prefix⟩ Contains Glob   [All types ▾]
→ SCAN 0 MATCH user:* COUNT 500
```

That one line of small print resolves the number-one confusion of every Redis GUI — "I definitely have this key, why can't I find it" (the answer is usually
that the user assumed substring search, while `MATCH` is glob matching). **Putting the generated command in front of the user works better than any hint text**,
and teaches the user how `SCAN` works along the way.

**The main view is a flat list, not a tree.** (Changed from a tree on 2026-08-18; reasons in §6.2.1.)
Each page SCAN returns is merged into the scanned set and the whole set is re-sorted, rendered virtualised, never waiting for "everything to load".
When automatic scanning reaches the soft cap (5000 keys or 3 seconds by default) it **stops** and suggests "results may be incomplete; consider narrowing the prefix",
leaving the decision to continue to the user — rather than scanning straight through a production database on their behalf.

**Key names are binary-safe.** Redis keys are byte strings and may not be valid UTF-8. The list shows them escaped the way redis-cli does
(`\xNN`), copying yields the escaped form (which can be pasted straight into the console), and `RENAME`/`DEL` use the **raw bytes**.
This needs a round-trip test to pin it down — most GUIs silently corrupt the user's keys right here.

#### 6.2.1 Why the tree became a flat list

The first version was a prefix tree (folded on `:`). On a real server it was "too painful" to use, and the pain was not about feel — **the tree was asserting something
the server never said**:

| Problem with the tree | Consequence |
|---|---|
| Redis keys are **flat byte strings**; `:` is only a writing convention | The tree draws a convention as a hierarchy |
| Each row shows only its own segment (`profile`), not the full key name | Knowing which key you're looking at means assembling the path in your head; copying a key name is especially painful |
| Deep chains of single children (`demo` → `user` → `10086` → `profile`) | Three clicks to see one key, and no level tells you anything |
| Indentation eats all the width | TTL / size / encoding — none of them fits, only a type badge is left |
| Counts are "scanned so far", but the tree looks like a complete directory | Users read `user (842)` as a definite value like `DBSIZE` |
| When `a:b` and `a:b:c` coexist, `b` has to split into a key node + a prefix node | What clicking it means is ambiguous — the price the tree shape is forced to pay |

The current shape: **one full key name per row** (the same as what redis-cli shows and what the code writes), with type / TTL / size as columns,
25–30 keys per screen. Hierarchy is demoted to two kinds of navigation:

- **Group rows**: keys with the same prefix fold into one row, `demo:order:2026:*  40`, only **once they reach the threshold** (8 by default, adjustable via `groupThreshold`);
  clicking it **expands in place**, with no nesting. The prefix is the batch's **longest common segment prefix**,
  so what folds is `demo:order:2026:*` rather than a vague `demo:order:*`. A handful of keys with the same prefix are always laid out flat
  — **folding is there to suppress noise, not to create clicks**; folding three keys into one row only means one more click to see them.
- **Breadcrumb**: `All › demo ›`; clicking a segment sets the filter bar to that prefix and rescans. Drilling down reuses the filter bar and **does not set up
  a separate navigation state** — otherwise "which batch of keys am I looking at" would have two conflicting sources, and the command echo would only know one of them.

The accompanying changes: type badges are **coloured by type** (a whole column in one colour carries no information; with forty `string`s in a row you can only read them one by one);
key names are **truncated in the middle** rather than at the end (truncating at the end eats the most distinctive segment); the left pane is widened from 280px to 400px
**and can be dragged** (real key names like `session:v2:prod:user:…:token` easily run to 40 characters); group rows **are not selectable**
(they are not keys; letting one carry the highlight would contradict the detail pane).

**The whole geometry and the state colours align with the host's Resource Explorer** (the session tree), item by item:

| | Resource Explorer | Redis key list |
|---|---|---|
| Group row height / indent | 30 / 12 | 30 / 12 |
| Group chevron | lucide `chevron-down`⁄`chevron-right` 12×12, `VelaTextTertiary` | Same |
| Group name / count | `FontSize12` Medium `VelaTextPrimary` / `FontSize10` `VelaTextTertiary` | Same |
| Leaf row height | 28 | 28 |
| Indent per level | group 12 → sessions in the group 36 (i.e. +24) | +24 |
| Leaf name | `FontSize11` monospace `VelaTextSecondary` | Same |
| Hover / selected | `VelaBgHover` / `VelaBgActive` | Same |
| Selected leaf name | `VelaAccent` + Medium | Same |
| Badge | corner radius 2, padding 6,1, `FontSize9` Medium | Same |

The chevrons are not the characters `▸▾` but lucide stroke paths: a `Viewbox` (12) wrapping a `Path` (24×24 view box,
`StrokeThickness=2`, round caps and joins), with the stroke width scaled proportionally to 1 — exactly the host's `LucideIcon` algorithm
(the 2/24 ratio). The plugin **does not reference** `VelaShell.Controls`; the geometry comes from the host theme via `{DynamicResource Icon.chevron-*}`,
the same route as the colour tokens.

**One deliberate misalignment**: the indent applies only to the key name, not the whole row. The Resource Explorer has no columns, so shifting the whole row right doesn't matter;
the three columns after the key list's name are **table columns**, and if they followed the indent they would never line up again — seeing the hierarchy doesn't mean the number columns may go crooked.

The folding rules are a pure function (`RedisKeyLayout`), pinned by 12 unit tests — folding wrongly doesn't throw an exception, it just "looks odd".

### 6.3 Type editors: each type the way it really is

The header of the detail pane on the right is the same for every type (key name / type badge / TTL / size / `OBJECT ENCODING` / action buttons);
the body changes with the type:

| Type | Body | How it is read | Key interactions |
|---|---|---|---|
| **string** | Value editor + encoding switch | `GET` (beyond the limit, `GETRANGE` fetches the first N bytes and marks the value truncated) | Auto-detects JSON (indented pretty-print + folding) / gzip / MessagePack / Java serialisation / hex; **decoded views are read-only by default**, and editing requires explicitly ticking "write back in decoded form" (otherwise one save turns a binary value into its text shape — a step that really does destroy data) |
| **hash** | Field table (virtualised, inline editing) | `HSCAN` (with `NOVALUES` when listing fields on 7.4+) | Field-level filtering goes through `HSCAN MATCH` (server-side filtering, not a front-end filter); on 7.4+ shows field-level TTL (`HTTL`/`HEXPIRE`) |
| **list** | Row table with indexes | `LRANGE` window paging | push/pop at both ends, `LSET` by index, `LINSERT`; **says plainly that "Redis has no primitive for deleting a middle element"** (you need `LREM` to delete by value), rather than pretending there is one |
| **set** | Member table | `SSCAN` | `SADD`/`SREM`, `SRANDMEMBER` to sample a few |
| **zset** | Member + score columns, sortable by score | `ZSCAN` / `ZRANGE ... WITHSCORES` | Edit scores directly (`ZADD`); query by score range and lexicographic range (`ZRANGEBYSCORE`/`BYLEX`) — that is how zsets are really used; merely listing members is pointless |
| **stream** | Entry table (ID / fields), with consumer groups on another tab | `XRANGE` paging, `XINFO STREAM/GROUPS/CONSUMERS` | View pending (`XPENDING`), `XACK`, `XADD`; consumer-group lag at a glance |
| **JSON** (module) | Path tree + value editing | `JSON.GET $ ` / `JSON.OBJKEYS` | Partial reads and writes by JSONPath; large documents are not fetched whole |
| **Unknown/module types** | Raw view | `TYPE` result + `OBJECT ENCODING` + `DUMP` length | Says explicitly "this type has no dedicated editor; you can work with it in the console", with a button that fills the key name into the console |

**The TTL editor** (shared by all types): the input accepts `900` / `15m` / `2h30m` / `2026-08-20 12:00` — three notations —
and echoes live below it "expires at 08-20 12:00:00 (2 days 3 hours left)"; `PERSIST` is an explicit button rather than clearing the input
(clearing is too ambiguous). In production, **adding** a TTL to a key that has none, or making a key with a TTL permanent, both go through a confirmation.


#### 6.3.1 Binary values: display and write-back use the same reversible representation

Like keys, Redis values are **binary-safe byte strings**. protobuf, msgpack, gzip, serialised objects —
what gets stored in Redis is often not text. Most GUI clients silently corrupt the data here:

> Decode as UTF-8 for display → the user clicks Save → encode as UTF-8 and write back.
> Invalid sequences were already replaced by replacement characters **at the decoding step**, so "Save" actually means
> "overwrite the original with an approximation", while the UI looks normal the whole way.

**The rule: display and write-back must use the same reversible representation, and the raw bytes are kept throughout.** The value editor offers three forms:

| Form | When it is available | Editable | Notes |
|---|---|---|---|
| **Text** | The bytes are valid UTF-8 (newline/carriage return/tab allowed) | ✓ | Save = `UTF8.GetBytes(draft)` |
| **Escaped** | Always | ✓ | redis-cli style `\xNN`, **reversible** — binary values can be edited byte-exactly too |
| **Hex** | Always | ✗ | A dump with offsets + an ASCII side column. It is **layout**, not a representation; editing on it would mean guessing which characters are data |

A few deliberate decisions:

- **For a non-UTF-8 value, the "Text" button is greyed out.** Not to restrict the user, but because that path goes through a lossy decode:
  the replacement characters on screen are not bytes actually in the value, and a casual save replaces the original with that approximation.
  To rewrite a binary value as plain text, just type it in "Escaped" (the escaped form accepts directly typed Chinese just as well).
- **Unrecognised escapes are always an error, never a guess.** Treating an unknown escape as a literal means changing the bytes the user meant to write
  without them noticing. On a malformed escape **not a single byte is written**, and the error position is stated.
- **Switching forms first decodes the draft back to bytes and then re-renders** — switching forms halfway through an edit does not lose the edit.
- **Newline/tab count as text.** Key names are always escaped (a key name with a newline would wreck the list's row height), but values can't copy that:
  showing a multi-line JSON as `{\n  "a": 1\n}` forces the user to edit, through escape characters, text they could have read directly.
- **A truncated value (beyond the preview limit) is always read-only** — letting the user edit "the first 256 KB" and write the whole thing back
  would silently delete the remaining few MB in a single save. This held before this change too.

The round trips are pinned case by case by `RedisValueTextTests`, one of which **specifically reproduces the pre-fix corruption path**
(escaped text encoded as ordinary text, turning a 10-byte gzip header into 40 bytes of ASCII);
there is also an end-to-end test against a real Redis: open a binary key → change one byte → save → ask the server directly for the raw bytes and compare.

**The member tables (hash fields / list items / set members) don't have these forms yet.** Their read/write path is still strings,
so for now it is **explicitly blocked**: as soon as a member's field name or value looks like the product of escaping, saving is refused with a hint to use the console instead.
Better to be temporarily unable to edit binary members in the member table than to let a save replace one with a string of literal backslashes.
Moving the member tables to "bytes + forms" as well is the next step; see §11.5.

### 6.4 The console: completion data comes from the server

The first tab of the bottom drawer, a real redis-cli:

- **The prompt reflects the connection state truthfully**: `10.0.3.12:6379[db0]>`; inside a transaction it becomes `…[db0](TX)>`,
  in subscribed state `…(subscribed:2)>`, and during MONITOR `…(monitor)>` with only a "Stop" button left.
  **Visible state = the user never wonders "why is my command doing nothing"** (in subscribed state RESP2 only accepts the subscribe family of commands).
- **Completion and argument hints come from `COMMAND DOCS` / `COMMAND INFO`** (7.0+, fetched once at startup and cached).
  This one step settles three things at once: the command table **always matches this server's version**,
  it **automatically includes module commands** (`JSON.SET`, `FT.SEARCH`, `TS.ADD` all come for free),
  and there is **no 240-row table to maintain in the plugin**. Below 7.0 it falls back to an embedded condensed table (about 120 common commands)
  and the hint says "this server provides no command documentation; completion uses the built-in condensed table".
- **Replies are rendered in redis-cli's format**: `(integer) 3`, `(nil)`, `(empty array)`, the `1) 1) "a"` indentation of nested arrays,
  RESP3's own notation for maps/sets. Error lines are prefixed `(error)` in `VelaError`.
  Binary replies get the escaped form + an expandable column with a decoded preview (like `\xe5\xbc\xa0\xe4\xb8\x89  张三` in the wireframe above).
- **Two-way linking**: any "key-looking" token in the console output is clickable → opens that key in the browser on the left;
  conversely, every key's context menu in the browser has "Generate command in console" (prefilling something like `HGETALL <key>`,
  with the cursor left at the argument). **This is the stitch that sews "click click click" and "type commands" together** —
  heavy Redis users always come back to the command line; the client's value is in **reducing the friction of going back and forth**, not in replacing it.
- **History**: `↑`/`↓` recall history (the same feel as the AI plugin's input box), stored in the plugin's private time-series store, still there when the document is reopened;
  `Ctrl+L` clears the screen, `Ctrl+C` interrupts a long stream (MONITOR/subscriptions).
- **Dangerous commands go through the gate** (§6.7), and **can't be fudged with the `→` echo**: the gate is a dialog, not a hint.

### 6.5 Overview / slow log / clients / subscriptions / memory analysis

The other tabs of the bottom drawer can all be "⤢ pulled out into a standalone document" too (one drag in VelaDock makes a split, to view it next to a terminal).

**Overview** (all `INFO` sections + `CONFIG GET`, polled every 5s by default, can be turned off):
four sparklines (ops/sec, hit rate, memory, connection count) + three summary cards:
persistence (time of the last RDB/AOF, `rdb_changes_since_last_save`, the reason for the last failure),
replication (role, the replica list with lag in bytes, `master_link_status`),
keyspace (key count and keys-with-TTL count per database, from `INFO keyspace`).
Every chart's data points go into the plugin's private time-series store, so **you can close the document, reopen it and still see the curve from a moment ago**
(the host's resource monitor already sets the precedent for accumulating time series).

> The database dropdown shows each database's key count directly (`db0 (1.2M)` / `db1 (empty)`), from `INFO keyspace`.
> One dropdown saves the blind testing of "click into each database to see whether there's anything in it" — a small change with a frequent payoff.

**Slow log**: a `SLOWLOG GET 128` table (duration / time / client / command); the command column is clickable → filled into the console;
`SLOWLOG RESET` and `CONFIG SET slowlog-log-slower-than` are offered right there (the latter goes through the write gate).
There is also a per-command latency ranking from `INFO commandstats` / `latencystats` (7.0+) — better than the slow log at showing global hot spots.

**Clients**: a `CLIENT LIST` table (addr / name / age / idle / cmd / db), sortable and filterable;
`CLIENT KILL` goes through a confirmation. The client's own connections are **highlighted and cannot be killed**
(a client killing itself and then reporting "connection lost" is a silly but very common bug).

**Subscriptions**: subscription management in the upper half (channels/patterns/shard channels), the message stream in the lower half (time / channel / payload, pausable, exportable).
There is also a one-click switch for "keyspace notifications" — it needs `CONFIG SET notify-keyspace-events KEA`,
so it **says explicitly that this changes the server configuration**, shows the value before the change, and restores it when switched off.

**Memory analysis** (a differentiating feature, the page operators need most):
sample a batch of keys with `SCAN` + pipelined `MEMORY USAGE`, and aggregate **by prefix** into
a tree of "which kind of key eats how much memory", plus a Top-N list of big keys (with type and element count).
The page is labelled throughout "sampled N keys, x% of the estimated total" — **it is a sampled conclusion, not a full audit**, and that has to be written on the page,
not hidden in the documentation. This page fills the gap left by `redis-cli --bigkeys` / `--memkeys`:
the command-line versions either block or only give type-level summaries, and are gone once you've read them.

### 6.6 Keyboard first (no conflicts with the host's existing bindings)

| Key | Action |
|---|---|
| `/` or `Ctrl+F` | Focus the filter box |
| `↑`/`↓`, `←`/`→` | Move within the tree, expand/collapse |
| `Enter` | Open the selected key |
| `F2` | Rename (`RENAME`; when the target already exists it asks explicitly "overwrite or cancel", corresponding to `RENAMENX`) |
| `Delete` | Delete (multi-select for batches; `UNLINK` preferred over `DEL` — asynchronous freeing does not block the instance) |
| `Ctrl+C` | Copy the key name (escaped form); `Ctrl+Shift+C` copies the fetch command |
| `Ctrl+R` | Rescan the current prefix |
| `Alt+1..9` / `Alt+0` | Switch database |
| `` Ctrl+` `` | Expand/collapse the bottom drawer |
| `Ctrl+Enter` | Run in the console (runs everything when there are several lines) |
| `Esc` | Close a menu / cancel a scan / leave MONITOR |

Three command-palette commands are registered as well (`Ctrl+P`): `Redis: Open connection…` / `Redis: Run command…` /
`Redis: Probe for Redis from the current SSH session` (see §7.1).

### 6.7 Safety guardrails: this is an ops tool, not a toy

Redis has no transaction rollback, and there is no recycle bin after `FLUSHALL`. The guardrails come in three tiers by "**cost of a mistake**":

| Tier | Commands | Handling |
|---|---|---|
| Write | `SET`/`HSET`/`DEL`/`EXPIRE`/… | In **read-only mode** they are stopped outright with an explanation (on by default in production, with a `[Read-only]` badge in the title bar; one click toggles it, with a confirmation). The criterion is the `write` flag returned by `COMMAND INFO`, **not a blacklist hand-written in the plugin** — if the server says it is a write command, it is one |
| Dangerous | `CONFIG SET`, `CLIENT KILL`, `SLAVEOF`/`REPLICAOF`, `SCRIPT FLUSH`, `ACL SETUSER`, `MONITOR`, `DEBUG` | A confirmation dialog every time, stating the concrete consequence (the `MONITOR` one must say "will noticeably reduce instance throughput", with an auto-stop countdown) |
| Destructive | `FLUSHDB`, `FLUSHALL`, `SHUTDOWN` | Requires **typing a confirmation string** (enter `prod-cache/db0`, the same as deleting a repository on GitHub). Under the production tag they are **disabled entirely by default**, and must be unlocked explicitly in the connection settings |

Three more rules:

- **`CLIENT NO-TOUCH on`** (7.4+, sent when probed). Browsing keys updates LRU/LFU metadata,
  mixing the client's paging into the server's eviction statistics — a read-only browser should not change what it observes.
- **No emptying on the user's behalf**. Deleting a prefix is a batched `UNLINK` (with truthful progress, interruptible), but it never quietly escalates
  "delete this prefix" into "empty this database" (the same rule as the S3 document's "only empty buckets are deleted; never empty one on the user's behalf").
- **The server is the last gate**. The settings page and the documentation must both say: real read-only access depends on ACLs
  (`ACL SETUSER viewer on >pw ~* +@read`); the client's read-only mode guards against **slips**, not malice.
  Stating the boundary is more responsible than implying "we guarantee safety".

### 6.8 Empty states, errors and degradation

| Situation | Presentation |
|---|---|
| The database is empty | One line in `VelaTextMuted`, "db0 has no keys", + a "New key" button, **not** an empty table |
| The prefix finds nothing | "Scanned 120,000 keys, none matched `user:*`" + the real `MATCH` pattern echoed + a one-click switch to "use Contains matching" |
| Command disabled / no permission (managed instance) | **An empty state, not an error**: for example the Clients page says "this server does not expose `CLIENT LIST`" and greys out the tab rather than showing a red error bar (the lesson of S3 §6, item 4, carried straight over) |
| Choosing a database on a cluster | The dropdown is disabled, with the note "Cluster mode only has db0" |
| The key expires while being viewed | The detail pane turns into "this key has expired or been deleted" in place, **without an error dialog** (this is the normal life cycle, not a failure) |
| The value exceeds the preview limit | A yellow bar at the top, "showing only the first 256 KB (of 4.2 MB)", + a "Save as file" button; the editor is set read-only so truncated content is never written back whole |
| Connection lost | Reuses the host's existing disconnect overlay spec (`VelaStatusDisconnected` + a Reconnect button); after a successful reconnect it **returns to the previous key and scroll position** |

---

## 7. Three things only VelaShell can do

Matching RedisInsight / Another Redis Desktop Manager feature for feature is the passing grade (§6 covers the bulk of their feature surface).
The real differentiation comes from **living inside an SSH client**:

### 7.1 Hook up Redis from an SSH session in one step (the killer feature)

The user is already SSHed into that machine. At this point the plugin can do what a standalone Redis GUI never can:

```
Ctrl+P → "Redis: Probe for Redis from the current SSH session"
  ├ RemoteExec:  ss -lntp | grep redis   → finds 127.0.0.1:6379 and :6380
  ├ RemoteExec:  redis-server --version / redis-cli -v
  ├ RemoteFs:    read /etc/redis/redis.conf → requirepass / port / bind / tls-port
  └ shows "Found 2 Redis instances" → tick → connection profiles created automatically (tunnelled through the current session), opened directly
```

**Zero typing.** No hostname to copy, no port to remember, no password to dig out of a config file, no tunnel to open.
This step uses only existing capabilities (`RemoteExec` + `RemoteFs` + the tunnel from §3.2),
and it is the first time "the plugin lives inside a terminal client" produces a **multiplier effect** rather than just sitting side by side.

> Rule: the `requirepass` that is read **must not be logged** and must not enter any Trace; it is persisted only through the host's
> `Secrets`/session-credential paths (consistent with the host's policy). The probe commands run in one batched exec, not typed on the remote one by one.

### 7.2 Side by side with the terminal

One drag in VelaDock: the Redis key tree on the left, `redis-cli --stat` or a `tail -f` of the application log on the right.
An Electron client can't do this (it doesn't know where your terminal is), and a web client even less.
"Change a key → immediately see the application's reaction in the log" is the actual troubleshooting move, rather than Alt+Tab between two windows.

### 7.3 An auditable command history

Every console command is written to the plugin's private time-series store (the `redis_console` measurement, tag = connection id).
Combined with the host's existing audit culture, "who typed what against the production database, and when" can be traced.
For team scenarios (which are also the commercial-licence scenarios) this is worth more than any single-machine GUI.

---

## 8. Data, strings and persistence

| Data | Where it lives | Notes |
|---|---|---|
| Connection profiles (host/port/mode/TLS/environment tag/…) | **Host** `session_profiles` | Through `ConnectionType.Plugin` + `PluginSettings`; hence Gist cloud sync, the session tree, recent connections and WinSCP-style import come for free |
| Password / TLS passphrase | **Host** AES-256-GCM | `IsSecret` fields take the same path as host passwords; the plugin sees them only at the moment of `OpenAsync` |
| Panel preferences (split ratio, drawer height, column widths, visible columns, default decoder) | Plugin `Storage` | One set per connection |
| Console history, metric samples | Plugin `TimeSeries` | Two measurements, `redis_console` / `redis_metrics` (the quota is 8, which is enough) |
| Favourite keys and commands | Plugin `Storage` | "The few keys I always look at" pinned to the top — operators' real habit |

**Strings**: the plugin ships its own `Loc.cs` (Chinese and English, falling back to English when missing), estimated at about 250 strings.
The reason is the same as for S3: making the host carry a five-language dictionary for a domain it doesn't even know is putting the cart before the horse.

---

## 9. Compatibility matrix and known pitfalls

| Target | Status | Notes |
|---|---|---|
| Redis 6.0 – 8.x standalone | Primary target | `HELLO 3` gets RESP3; below 6.0 falls back to RESP2 (when `SCAN TYPE` is unavailable, a `TYPE` pipeline fills in) |
| Redis Cluster | Supported | db0 only; `SCAN`/`DBSIZE`/`INFO` per node; `MOVED`/`ASK` redirects; cross-slot commands (multi-key `MGET`/`DEL`) must be grouped by slot, or `CROSSSLOT` reported truthfully |
| Sentinel | Supported | The master address comes from `SENTINEL get-master-addr-by-name`; subscribing to `+switch-master` detects failover and reconnects automatically |
| Valkey / KeyDB / Dragonfly | Best effort | The command surface is largely compatible, but `INFO` fields, `OBJECT ENCODING` values and module support all differ → the capability probing of §5.3; a missing `INFO` field is always shown as "—", never 0 |
| Managed services (the clouds / Upstash) | Best effort | `CONFIG`/`CLIENT`/`DEBUG`/`SLOWLOG`/`MONITOR` are often disabled or renamed → degrade to empty states (§6.8) |
| TLS (including self-signed) | Supported | The same "prompt → remember the fingerprint → reconnect" as FTPS/S3; **never wait synchronously inside the TLS callback for the user to click a button** (that turns an asynchronous dialog into a blocking one and deadlocks very easily — the S3 work hit this) |
| Redis 5 and earlier | No promises | No ACLs, no `COMMAND DOCS`, different `MEMORY USAGE` semantics; connect if it connects, with features degraded by probing |

**Ten pitfalls you are bound to hit** (written into the implementation checklist, each with a matching test):

1. **`KEYS` may not be used even once** — including the temptation to "just count the total" (use `DBSIZE`).
2. **`SCAN` returns duplicate keys** (during rehashing), and may return empty pages for several rounds in a row; only a cursor of 0 means the end. The UI deduplicates.
3. **`SCAN COUNT` is a hint, not a guarantee**; the number returned can be anything from 0 to far more than COUNT.
4. **Keys and values are both binary**; UTF-8 cannot be assumed. Escaped display + operations on the raw bytes.
5. **`MEMORY USAGE` is a sampled estimate for collection types**; the UI marks it "approx.".
6. **The only ways out of `MONITOR` are `RESET` (6.2+) or disconnecting**, and the throughput loss is significant, so there must be an auto-stop.
7. **A connection in subscribed state (RESP2) only accepts the subscribe family of commands** — so subscriptions need their own socket, or confirmation of RESP3.
8. **`RENAME` silently overwrites the target**; only `RENAMENX` doesn't. A GUI must ask.
9. **`DEL` on a large collection blocks** — always prefer `UNLINK` (4.0+).
10. **`SELECT` fails outright on a cluster**; the concept of `db` does not exist in a cluster, so the UI must disable it up front rather than wait for the error.

---

## 10. Test strategy

Following the approach the repository has already proven (what worked for `LoopbackS3Server` works here too, and more cheaply — RESP is easier to fake than XML):

| Test | Count (est.) | What it guards |
|---|---|---|
| `RespCodecTests` | ~40 | Read/write round trips for all 14 RESP2/RESP3 types, fragmented arrival (correct even when fed one byte at a time), very long bulk strings, the three ways of writing null |
| `LoopbackRedisServer` + `KeyspaceServiceTests` | ~35 | End to end: SCAN paging/duplicate keys/empty pages/cursor back to zero, both paths with and without `SCAN TYPE`, incremental tree building, binary key round trips, batched `UNLINK` progress |
| `ClusterRoutingTests` | ~20 | CRC16 slots (checked against Redis's official samples), hashtags, `MOVED`/`ASK`+`ASKING` retries, grouping cross-slot commands |
| `SentinelTests` | ~10 | Master address discovery, `+switch-master` triggering a reconnect |
| `SafetyGateTests` | ~20 | Read-only mode stopping write commands by the `COMMAND INFO` flags, the three gate tiers, confirmation-string checks, `FLUSHALL` disabled entirely under the production tag |
| `ValueCodecTests` | ~20 | Detecting JSON/gzip/MessagePack/Java serialisation; **a decoded view must not turn a binary value into its text shape** (the path in §6.3 that really destroys data) |
| `DegradationTests` | ~15 | Disabled/renamed/unauthorised commands always end in an empty state, never an error (simulating managed instances) |
| Headless panel UI | ~20 | Key-tree virtualisation, the filter bar's command echo, TTL input parsing, drawer tab switching |
| Host-side `PluginWorkspaceTests` | ~15 | Declare → register → lazily activate → unregister, the `isolated` combination rejected, document life cycle, tunnel lease torn down when the document closes |

Acceptance on real servers (the parts a loopback server cannot verify): bring up in Docker a standalone Redis 7.4 + a cluster of 3 masters and 3 replicas + Sentinel,
and run connect/scan/write/failover once on each; also take one managed instance through the degradation paths.
`docker-compose.test.yml` already sets the precedent; two more services are enough.

---

## 11. Milestones and effort

| Milestone | Contents | Estimate |
|---|---|---|
| **M0 Host extension** ✅ | `contributes.workspaces` + `IWorkspacesApi` + `PluginWorkspaceDocument` + registry/lazy activation/exception translation + 25 host-side tests. **The declarative SSH tunnel (`ProtocolSettingKind.SshSession`) was not done and moved to M4** | 3–5 d |
| **M1 Connect and view** ✅ | Connection (SE.Redis), capability probing, cursor-SCAN key tree, read-only views + paging for five types, status strip, Chinese and English strings, 72 tests (including 21 against a real server and 7 headless panel tests) | 5–7 d |
| **M2 Edit and type** ✅ | Editors for every type, TTL editing (with live conversion echo), rename (no silent overwrite)/delete, the console (server-side completion/cross-session history/redis-cli-style reply rendering), the three guardrail tiers, the read-only switch | 6–8 d |
| **M3 Ops surface** ✅ | Overview (five groups of INFO metrics), slow log, clients (own connections cannot be killed), Pub-Sub, sampled memory analysis. **The metric time-series charts and the keyspace-notification switch were not done** (see §11.2) | 6–8 d |
| **M4 Differentiation and wrap-up** ✅ | Declarative SSH tunnel (carried over from M0), the §7.1 SSH probe + connection proposals, the `DUMP`/`RESTORE` cross-database copy primitive, favourites, console history in the time-series store, acceptance on real servers. **The dedicated JSON module editor was not done** (see §11.2) | 6–8 d |

About **5–6 weeks** in total (one person, including tests and documentation), roughly 10–11k lines of code, the same order of magnitude as the AI assistant plugin.
At the end of M1 it is already usable day to day ("viewing" is the vast majority of everyday Redis work); at the end of M2 a preview can be published.

**Suggested order of entry**: do M0 first and **get M1's connection layer running at the same time** — because M0's design is only proven correct once a real consumer
has validated it (which is also the original intent of `docs/plugins/STATUS.md` §5, "write the first business plugin to feed back into the framework's gaps").

### 11.1 M0 + M1 shipped (2026-08-17)

They were done together as suggested above, and in hindsight that was right: both design flaws in M0 were pushed out by M1 as a real consumer
— the `Declare` overloads were ambiguous at `[new() { … }]` (renamed `DeclareWorkspaces`),
and "re-registering on a language switch kills already-open tabs" (the provider must reuse the same instance).
Neither would have been thought of at the pure design stage.

| Deliverable | Location |
|---|---|
| SDK workspace capability domain | `plugin-sdk/VelaShell.PluginSdk/Workspaces/` (descriptor / request / document / provider / capability surface), reusing the protocol side's `ProtocolSettingField` and the `Protocol*` exception family |
| Manifest and activation | `contributes.workspaces` + `onWorkspace:` + the `isolated` combination rejected + a clash with a protocol id rejected |
| Host registry | `PluginProtocolRegistry` generalised into a connection type registry (with `Kind`); both shapes share the tab strip and the lazy-activation chain |
| Host session | `PluginWorkspaceLauncher` (resolve → assemble the request → translate exceptions → register the session) + `PluginWorkspaceDocument` (the dock tab shell) |
| Redis plugin | `plugins/VelaShell.Plugin.Redis` (about 2.4k lines): connection, capability probing, cursor scanning, key tree, details for five types, declarative form, Chinese and English strings |
| Tests | 25 on the host + 72 in the plugin (51 pure unit tests + 21 against a real server + 7 headless panel tests); all 1989 tests in the repository green, 0 warnings |

**M1's limits**: values read-only, drawer not started, guardrails shown only as badges — all three have since been filled in by M2/M3; see §11.2.

### 11.2 M2 + M3 + M4 shipped (2026-08-17, the same day)

| Deliverable | Location |
|---|---|
| Command gate | `RedisCommandGuard`: the three tiers are classified **by the `COMMAND INFO` flags** (unknown means write); the "destructive" tier is fixed by name |
| Write path | `RedisConnection.Writes.cs`: strings (`KEEPTTL` by default), hash/list/set/sorted-set members, `EXPIRE`/`PERSIST`, `RENAME` (`RENAMENX` by default), batched `UNLINK`, `DUMP`+`RESTORE` cross-database copy |
| Console | `RedisConnection.Console.cs` + `RedisCommandLine` + `RedisReplyFormatter` + `Ui/RedisConsoleViewModel`: redis-cli-style reply rendering, server-side completion, cross-session history, unavailable commands **explained before they are typed** |
| Confirmation gate | `Ui/RedisConfirmation`: attached to this panel; the "destructive" tier requires typing the endpoint string |
| Ops surface | `RedisConnection.Ops.cs` + `Ui/…Drawer.cs`: five overview groups, slow log, clients (own connections cannot be killed), Pub-Sub (the library's dedicated subscription connection), sampled memory (aggregated by prefix + Top-N) |
| Host: declarative SSH tunnel | `ProtocolSettingKind.SshSession` + `MainWindowViewModel.EstablishWorkspaceTunnelAsync`: reuses an already connected session to the same target, otherwise connects through `IConnectionWorkflowService`; the local port is borrowed with bind-0; closing the document tears the tunnel down |
| Host: connection proposals | `IWorkspacesApi.ProposeConnectionAsync` → `PluginProtocolRegistry.ConnectionProposalHandler` → the host's New Connection dialog, prefilled. **The plugin cannot write the session store itself** |
| SSH probe | `RedisDiscovery`: one exec fetches listening ports/processes/versions/config paths, reads `requirepass` through `RemoteFs`, and proposes connections one by one |
| Private persistence | `RedisStore`: favourites in `Storage`, console history in `TimeSeries` (delivering the auditability of §7.3); degrades silently when there is no DB |
| Tests | 179 in the plugin + 25 on the host; all **2096 tests in the repository green, 0 warnings** |

**This round the tests caught three more real bugs** (none of which a compiler could see):
① `INFO everything` on 3.x replies with empty content **without an error** — the fallback condition has to look at "how many fields were parsed", not "whether the result is null";
② the copy for turning off read-only in production promised a typed confirmation string, but the code didn't require one — **copy that lies** is worse than not having the protection at all;
③ the `Loc` table had a duplicate key, which throws `ArgumentException` **during static construction** — the compiler doesn't care, and when it blows up the whole plugin is unusable (now guarded by `LocTests`).

**Explicitly not done (not oversights)**:

| Item | Notes |
|---|---|
| Overview metric time-series charts | The numbers are all there, but no curves are drawn and no time-series samples stored. Doing it first requires deciding how polite the sampling frequency is to a production instance |
| Keyspace-notification switch | It needs `CONFIG SET notify-keyspace-events`, i.e. changing the server configuration — worth its own design round (the value before the change, restoring it when switched off) |
| Dedicated editors for the JSON / Search / TimeSeries modules | Reachable through the console for now; the overview doesn't list loaded modules either |
| The stream consumer-group tab | Stream entries can be viewed; `XINFO GROUPS`/`XPENDING` aren't in the UI yet |
| Cluster / Sentinel | The code paths are written, but **verified only on a standalone server** — before going live they need a run each on 3 masters + 3 replicas and on Sentinel |
| Persisting panel preferences | Split ratio, drawer height and column widths are not remembered yet (planned in §8) |

### 11.3 Four fixes after running the UI on a real server (2026-08-18)

Starting the program for real and connecting to a local Redis 3.0.504 along the normal path surfaced four problems that only show up on a real machine:

| Symptom (what the user sees) | Root cause | Fix |
|---|---|---|
| The tab reads `VelaShell.Docking.PluginWorkspaceDocument` | `DockGroupControl` had no `DataTemplate` for `PluginWorkspaceDocument`; Avalonia **reports no error** and falls back to `ToString()` | Added `WorkspaceDockTabItem` (the same status dot/accent colour/context menu as the SFTP tab) + registered the template; the document exposes an observable `Status`. Regression test: `WorkspaceDockTabUiTests` |
| The key tree is blank and the status strip says `A target database is required for SCAN` | The cluster path went through `IServer.ExecuteAsync("SCAN", …)`, and **that path carries no database number** | Switched to `IServer.KeysAsync(database, pattern, pageSize, cursor)` (the library's proper route for "SCAN on this node", which takes the database number explicitly and whose enumerator implements `IScanningCursor`); the cost is no `TYPE` option, so the cluster always degrades type filtering to the client. Regression test: `ClusterDeploymentAgainstStandalone_StillScans…` (pinned down on a standalone server too) |
| Pressing "Test" waits a few seconds, then reports `The connection could not be established - Timeout` | The host's `TestConnectionAsync` **unconditionally opened an SSH connection** — connecting to 6379 with SSH, TCP gets through but hangs at version exchange. S3 was hit the same way | `IConnectionWorkflowService.PluginProbe`: for plugin connections the UI layer now really opens a plugin session once and closes it (the workspace shape builds/tears down the tunnel along with it, the file-system shape opens and closes a session once); when no probe is attached it says plainly "this connection type can't be tested", **giving no fake reason** |
| Pressing "Test" looks as if nothing happened | The test result was rendered at the **very bottom of the scroll area**, while the button is in the footer — with a dozen or so Redis fields it always scrolls, so the result lands out of view | The feedback bar was moved out of the `ScrollViewer` and pinned just above the button row (`HasFeedback` controls the space) |

One more place where "copy patched over the UI" was turned into a declaration: `ProtocolSettingField.VisibleWhen`.
"Master name" used to be shown in Standalone/Cluster mode too, relying on a line of small print under the field saying "Sentinel mode only";
a "Default database" filled in under Cluster meant nothing, also explained in small print, "a cluster only has db0". Now each of the two fields declares its own visibility condition,
and simply doesn't appear when it doesn't apply (the value is still kept and passed back). The predicate's shape is deliberately closed to "key ∈ set of values"; the reason is in §12.

**This round the tests also caught a bug I had just written myself**: the mode-mismatch warning used to be written into `StatusMessage`,
and that line is cleared on every scan — the warning was wiped by the next scan as soon as it was written, as if it had never been said.
It became a persistent `DeploymentWarning` bar instead (a configuration error doesn't fix itself; it needs a place of its own).

All **2102 tests in the repository green, 0 warnings**.

### 11.4 The key browser becomes a flat list (2026-08-18)

User feedback: "the key display on the left is too painful to use". The diagnosis and the new shape are in §6.2.1 — in short: the tree was asserting something
the server never said (Redis keys are flat byte strings; `:` is only a convention), and the price it paid was not seeing full key names,
a click tax on deep chains, and no room for TTL/size.

| Deliverable | Location |
|---|---|
| Folding rules (pure function) | `Ui/RedisKeyLayout.cs`: partition by the next segment → below the threshold lay out flat, at the threshold fold into one row; the prefix is the longest common **segment** prefix; the top-level common prefix never folds (it is the breadcrumb); expanding applies the same rules recursively to the members |
| Row model | `Ui/RedisKeyRow.cs`: two states, key row/group row, with type, TTL, size, indent, and a boolean for colouring by type |
| Incremental sync | `RedisKeyLayout.Sync`: reuses row objects by id, so re-sorting mid-scan doesn't knock out the selection or the scroll position |
| Batched measurement | `RedisConnection.MeasureAsync`: one pipelined round trip each for a page's TTL and size (**not including** `MEMORY USAGE` — it is a sampling traversal and doesn't exist below 4.0) |
| Breadcrumb | `Ui/RedisBreadcrumbSegment.cs` + `NavigateCommand`: drilling down reuses the filter bar, with no separate navigation state |
| New setting | `groupThreshold` (8 by default, advanced) |
| Style alignment | Row height/indent/font size/chevrons/hover and selection colours aligned item by item with the host's Resource Explorer (see the table in §6.2.1); chevrons via `{DynamicResource Icon.chevron-*}` + `Viewbox`+`Path`, replicating `LucideIcon`'s 2/24 stroke ratio, **without referencing** `VelaShell.Controls` |
| Tests | 12 in `RedisKeyLayoutTests` (pure function) + 1 `KeyList_FoldsTheNoisyPrefix…` (end to end against a real Redis) |

**Removed**: `Ui/RedisKeyNode.cs` and `RedisKeyTreeTests.cs` (the tree is gone, and its model went with it);
the two strings `Redis_TreeCountTip` / `Redis_ColumnCount`.

The real server caught two more things that only show when it's running:
① clicking a group row let the ListBox grab the highlight while the detail pane stayed on the previous key — vetoing in the view model wasn't enough;
   `ListBox` ignores notifications during its own assignment, so putting the selection back had to be queued for the next round;
② after putting it back, the list **scrolled along** to the selected item, which looked like "click a group at the top, the view jumps to the bottom" —
   `AutoScrollToSelectedItem` was turned off, with an explicit `ScrollIntoView` only on the "jump to a favourite key" path.

All **2112 tests in the repository green, 0 warnings**.

> **Not verified**: the cluster SCAN fix has only been run on a standalone server (there is no cluster environment on this machine, and a production cluster is deliberately not used for experiments).
> Before going live it still needs a rerun on a real 3-master, 3-replica setup — advancing the cursor node by node, resuming the scan across nodes, the `node|cursor` cursor encoding.

---


### 11.5 Binary handling in the value editor (2026-08-18)

It started with a question: "what does the value editor do with binary data". Digging in turned up not a missing feature but **a data-corruption path
that was live**: values were decoded as UTF-8 for display and encoded back as UTF-8 on save. The design is in §6.3.1.

| Deliverable | Location |
|---|---|
| Bytes ↔ text (pure functions) | `RedisValueText`: `IsTextSafe` / `Escape` / `TryUnescape` / `HexDump` / `Detect` |
| Three forms | The view model holds the raw bytes in `_valueBytes`; `ValueFormat` decides the rendering; saving decodes back to bytes per the current form |
| Refusing writes | On a malformed escape **not a single byte is written**, and the error position is shown; hex is read-only |
| Member tables | Editing of binary members is blocked for now (see below), rather than doing a half-baked write-back |
| Tests | 14 in `RedisValueTextTests` (including the full 0..255 byte round trip and the one that **reproduces the corruption path**) + 3 end-to-end tests against a real Redis |

**An everyday problem fixed along the way**: any string value containing newlines (multi-line JSON, log snippets) used to be shown as
literal `\n` — because values reused the text check for **key names**, and that rule treats every control character as non-displayable.
That's right for key names (a key name with a newline would wreck the list's row height), but not for values. Now newline/carriage return/tab count as text.

**Explicitly not done**: the member tables (hash fields / list items / set and sorted-set members) still read and write strings.
The signatures at the `RedisConnection.Writes` layer are all `string`, and `RedisElement` only carries text.
Doing it right means changing both of those to bytes and giving the member tables the same form switch — a separate round of changes.
Until then, the member tables **refuse to write** binary members and suggest the console instead: one fewer edit allowed is something the user can notice,
one more corrupted write is something the user never sees.

All **2129 tests in the repository green, 0 warnings**.

---

## 12. Explicit out-of-scope boundaries

| Item | Why not |
|---|---|
| **RDB / AOF file parsing** | That is the job of offline forensic tools (the `redis-rdb-tools` kind), a different product from "connect to a live instance and operate on it". If you really need to look at an RDB, start it with `redis-server` and connect |
| **Data migration / sync jobs** | Long-running tasks, resumable transfers, consistency checks — that is the territory of `redis-shift`/`RIOT`. The client only offers "copy these selected keys to another connection" (`DUMP`+`RESTORE`, synchronous, interruptible, with a clear scale) |
| **`SYNC` / `PSYNC` full replication** | A client pretending to be a replica would pull the entire dataset; one misclick is one production incident |
| **A visual Lua script IDE** | `EVAL` in the console is enough; building an IDE is the size of another product |
| **A full-featured RedisSearch query builder** | v1 only does "the indexes are visible and you can type `FT.SEARCH`"; a query builder waits for real demand |
| **Isolated process mode** | The same hard constraint as the protocol capability: the host asks the plugin for an Avalonia `Control` to host in the dock area, and embedding native controls across processes has been dropped. The cost is the same as S3's (a plugin crash affects the host), acceptable for first-party plugins built from the same source |
| **Migrating old data** | There is no historical data to migrate — this is a brand-new connection type |

---

## 13. Open questions (need a decision)

1. **Should M0 be done together with this plugin** (this document suggests yes; reasons at the end of §11).
2. **Read-only mode on by default under the production tag** — will first-time users feel "why can't I change anything"?
   This document leans towards on by default + a prominent title-bar badge + a one-click toggle (with confirmation); the cost of the opposite is one accidentally deleted production key.
3. **Should `SELECT` be linked between the console and the browser**. This document leans towards **linking with an explicit notice**
   ("the console switched to db3; the browser followed"), because silent divergence is the worse failure mode.
4. **The initial scope of module support**: suggest doing only JSON (the most used, and where an editor adds the most value),
   with Search/TimeSeries/Bloom reachable only through the console + the list of loaded modules shown in the overview.
