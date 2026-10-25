# 08 · 失败分类与可观测性

> 对应实现：`Diagnostics/`（L9）。
>
> 本文件定义的是**契约**，不是实现细节：
> 使用者据此写 `catch` 与 UI 文案，所以这里的每一个取值一旦发布就不能随意改。

---

## 一 原则：异常里装数据，不装拼好的句子

一个失败必须能回答四个问题：

1. **是什么失败了？** → `SshFailureReason`（强类型枚举）
2. **在哪一步失败的？** → `SshPhase`
3. **对端说了什么 / 我方提供了什么？** → 结构化上下文（算法名单、尝试记录、状态码）
4. **下一步能做什么？** → `IsRetryable` + 上下文里的可操作信息

**禁止**把这些信息只写进 `Message` 然后让调用方去切字符串。
理由很直接：那正是我们要消灭的东西之一 ——
底层库把失败原因藏在 internal 异常里、只在消息前缀露一截，
上层为了区分「认证失败 / 超时 / 协商失败」只能去 `IndexOf(" - ")`。
那种代码在库升级改了一个字的措辞时会静默失效。

`Message` 是给人看的，**不是** API。

〔决策〕**比原因码更细的种类用 `*Kind` 枚举交出，零值一律是 `Unknown`（没归类）**：`SshChannelException.FailureKind`（`SshChannelFailureKind`）、
私钥与证书异常的 `ProblemKind`（`SshKeyProblemKind`）、认证尝试记录的 `SshAuthDetailKind`（[04 §3.4](04-authentication.md)）、主机证书的 `HostCertificateProblemKind`（[03 §5.5](03-key-exchange.md)），
建连异常的 `SshConnectException.FailureKind`（`SshConnectFailureKind`：主机密钥被拒的出处、代理与跳板的哪一步，§二、§5.2）、转发器起不来的 `SshForwardException.FailureKind`（`SshForwardFailureKind`，[07 §八](07-forwarding.md)）、
agent 的 `SshAgentException.FailureKind`（`SshAgentFailureKind`）、SFTP 起不来的 `SftpUnavailableException.FailureKind`（`SftpUnavailableKind`，[06 §一](06-sftp.md)），
以及 `SftpException.MissingExtensions`（缺的是哪个扩展）与 `ForwardErrorEventArgs` 的参数（[07 §五](07-forwarding.md)），
都连同各自的参数（路径、算法名、Key ID、上限……）一起交出。调用方按种类写界面文案；`Message` 仍是写给开发者的中文，措辞不承诺稳定。

〔决策〕**对端给的文本进 `Message` 之前先清一遍**：控制字符（`ESC`、`CR`、`BEL`…）、`DEL`、C1 控制码与双向文本控制符
一律换成 `?`，并截到固定长度。版本标识串、`DISCONNECT` 的描述、拒绝开通道的理由、SFTP 的状态消息、
代理的应答都来自对端，很多时候来自一个还没被认证的对端 —— 而 `Message` 最后会被打到终端或界面上，
原样拼进去就是一个转义序列注入面（改剪贴板、清屏伪造提示、盖掉前半行）。
**原话照样放在专门的属性里**（`PeerDescription`、`ServerMessage`），那些属性按不可信文本对待。

多行的对端输出（远端命令的 stderr）**只带末尾**：`SshCommandResult.EnsureSuccess` 的消息里放 stderr 最后 1024 个字符的清洗摘要，
换行收成一个「 ⏎ 」接成一行（一个换行就能在日志里伪造一行别的记录），出错的那一句通常在最后；
对端给的信号名同样清洗、截到 32 个字符。原文完整地留在 `SshCommandFailedException.Result` 里。

〔决策〕**这套规则公开出来**（`VelaShell.Ssh.Diagnostics.PeerText.Sanitize` / `SanitizeTail`）：使用者要把原话（`ServerMessage`、
`PeerDescription`、SFTP 的路径）自己拼进界面文案时用同一套，不必各写一份 —— 宿主曾经在已经清洗过的消息后面再追加一遍原文，
同一句话显示两遍，第二遍绕过了清洗。

---

## 二 异常层级

```
SshException                          抽象基类；带 Reason / Phase / IsRetryable
├── SshConnectException               建连阶段（拨号、版本、协商、主机密钥）—— 带 Hops
│   ├── SshNegotiationException       算法协商失败 —— 带双方名单
│   └── SshKeyExchangeException       密钥交换的计算失败（对端公开值不合法）
├── SshAuthenticationException        认证失败 —— 带逐方法尝试记录
├── SshProtocolException              对端违反协议
├── SshConnectionClosedException      连接已断（对端关闭 / 保活判死 / 收到 DISCONNECT / 本端中止 / 重协商时换了主机密钥）
├── SshChannelException               通道打不开，或通道上的请求被拒 —— 带原因码、种类与对端原文
├── SshCommandFailedException         远端命令没有成功结束（EnsureSuccess）—— 带完整结果
├── SshForwardException               转发器起不来
├── SshPublicKeyException             公钥 blob / 文本解析失败、类型不支持
├── SshPrivateKeyException            私钥文件读不出来、解不开 —— 带 NeedsPassphrase 与种类
├── SshCertificateException           证书读不出来，或与私钥不是一对 —— 带种类与两把公钥
├── SshAgentException                 连不上 agent，或 agent 拒绝了
├── SftpException                     SFTP 操作失败 —— 带 StatusCode 与服务端原文 ServerMessage
├── SftpTransferInterruptedException  传输中断 —— **带 DurableLength**
└── SftpUnavailableException          SFTP 子系统起不来
```

下面这些类型只对应一种失败，`Reason` 与 `Phase` 是固定的：

| 类型 | `Reason` | `Phase` |
| --- | --- | --- |
| `SshNegotiationException` | `NegotiationFailed` | `KeyExchange` |
| `SshKeyExchangeException` | `ProtocolError` | `KeyExchange` |
| `SftpUnavailableException` | `Unsupported` | `Open` |
| `SshCommandFailedException` | `CommandFailed` | `Open` |

下面这些 `Phase` 固定，`Reason` **按实情报**：

| 类型 | `Reason` | `Phase` |
| --- | --- | --- |
| `SshPublicKeyException` | `KeyFormatInvalid`、`Unsupported`（类型不支持） | `None` |
| `SshPrivateKeyException` | `KeyFileUnreadable`、`KeyFormatInvalid`、`KeyPassphraseRequired`、`KeyPassphraseIncorrect`、`Unsupported` | `None` |
| `SshCertificateException` | `KeyFileUnreadable`、`KeyFormatInvalid`、`KeyMismatch`、`Unsupported` | `None` |
| `SshAgentException` | `AgentNotRunning`、`AgentUnavailable`、`AgentRefused`、`LimitExceeded`（要加的钥超出报文上限）、`ProtocolError` | `Authenticating` |
| `SshChannelException` | `ChannelOpenFailed`（带 `OpenFailureReason`）、`ChannelRequestRejected`、`LimitExceeded`（本端的通道数 / 会话窗口总预算到了上限）、`ProtocolError`（对端宣告的 max packet 是 0，写进 stdin 的发不出去） | `Open` |
| `SshForwardException` | `ForwardRejected`、`ForwardBindFailed`、`ForwardSetupFailed`、`LimitExceeded`、`ProtocolError`；agent 转发因本机 agent 连不上而没开成时沿用 agent 那边的 `AgentNotRunning` / `AgentUnavailable` | `Open` |
| `SftpTransferInterruptedException` | 随中断的原因（内层异常）：断线 `ClosedByPeer`；服务端拒写（磁盘满、配额、权限）照内层 `SftpException` 的原因码；调用方取消、本端释放 `Aborted`；关闭时等确认超时 `Timeout`。〔决策〕曾经固定为 `ClosedByPeer`，按原因码判断的调用方会把「磁盘满」当成断线、照样去续传 | `Open` |

比 `Reason` 更细的种类（§一）：`SshPrivateKeyException` / `SshCertificateException` 带 `ProblemKind`（`SshKeyProblemKind`，两者共用）、
`Origin`（文件路径、`Parse` 的 origin 或密钥库里的钥名，由入口在异常离开库之前补上）与 `AlgorithmName`，证书另有 `KeyId`、
`CertificateKey`、`SignerKey`（配错时两边的公钥，界面据此显示指纹）；「文件坏了」的几十种细分合并成 `Malformed`，细节留在内层消息里。
`SshChannelException` 带 `FailureKind`（`SshChannelFailureKind`）与 `ChannelType` / `SubsystemName` / `Limit`，种类见 [05 §2.3](05-connection.md)。
建连异常 `SshConnectException` 带 `FailureKind`（`SshConnectFailureKind`）与 `Via`（跳板 / 代理 / ProxyCommand 命令行）/ `Target`（`host:port` 文本）/ `HostKeyType` / `HostKeyAlgorithm` / `HostKeyBits` /
`MinimumHostKeyBits` / `ProxyReplyCode` / `ProxyReply`（主机密钥类型与 HTTP 状态行来自对端、未清洗）；转发器起不来的 `SshForwardException` 带 `FailureKind`（`SshForwardFailureKind`）与 `Endpoint` / `Port` / `Limit` /
`Timeout` / `DisplayName` / `ExitCode` / `ErrorOutput`（xauth 的 stderr，已清洗）/ `ConfigLine`；`SshAgentException` 带 `FailureKind`（`SshAgentFailureKind`）与 `Endpoint` / `PipeOwner`（命名管道属主的 SID）/
`RequestedAlgorithm` / `SignedAlgorithm`（来自 agent，未清洗）/ `HasDestinationConstraints`；`SftpUnavailableException` 带 `FailureKind`（`SftpUnavailableKind`）与 `ServerVersion`。
这几个枚举都放在各自的领域文件夹，建连的那个在 `Diagnostics/`（底座不依赖别的文件夹，所以参数一律是字符串与整数）；异常类型与 `Reason` 不变，按类型 `catch` 的调用方不受影响。
TCP 拨号认不出的套接字错误原因码仍是 `Unknown`，种类 `TcpConnectFailed` 并带 `Target`，错误码在内层的 `SocketException`。

读私钥、读证书、解析公钥发生在连接之外（也可能根本没有连接），所以 `Phase` 是 `None`。
密钥交换里解析对端的主机密钥失败时，由密钥交换把它包成 `Reason` 为 `HostKeyRejected` 的 `SshConnectException`。

〔决策〕**`Reason` 必须说真话，不许写死。**曾经私钥、证书、agent、转发的异常一律报 `Unsupported`，
exec / pty-req / shell 被拒报成 `ChannelOpenFailed`（通道其实开成功了），跳板成环报成可重试的 `ProxyRefused`。
调用方于是只能去解析消息句子 —— 而消息是写给开发者的诊断文本，不是界面文案，也不承诺措辞稳定。
没有合适的值就加一个，不借一个相近的；只有异常类型本身只对应一种失败时才在类型里固定（上面第一张表）。

主机密钥被拒没有专门的类型：仍是 `Reason` 为 `HostKeyRejected` / `HostKeyChanged` 的 `SshConnectException`，**`FailureKind` 分开库自己拒的与策略拒的**：
`HostKeyMalformed`（`K_S` 解析不了）/ `HostKeyAlgorithmMismatch`（类型串与协商出的算法不符）/ `HostKeyTooShort`（RSA 不足下限，`HostKeyBits` / `MinimumHostKeyBits`）/ `HostKeySignatureInvalid`（交换哈希的签名验不过）
发生在问策略之前，消息是库的；`HostKeyPolicyRejected` 时 `Message` 就是策略给的原因（库不改写）；`HostKeyPolicyRejectedWithoutReason` 是策略返回了未填的裁决（`default`）。
〔决策〕曾经调用方只能假定「`HostKeyRejected` 的消息都是策略写的」，签名验不过时界面显示的是库的中文。

〔决策〕**公开 API 只抛上面这一层的异常。**对端发来的东西解不开（KEXINIT 的名单被截断、收到空载荷的帧）在握手、认证期间与会话期间
同一个口径：`SshProtocolException`（`ProtocolError`，`Phase` 是出事的那一步）；对端在一个报文中途断开是 `SshConnectionClosedException`
（`ClosedByPeer`）。解析层的异常是 internal 的，挂在内层。曾经握手期没有这层归类，它们原样漏出 `ConnectAsync` —— 调用方 `catch (SshException)` 接不住。
空载荷的帧在帧层就拒收（spec/01 §5），不交给上层。

〔决策〕**通道级失败不派生自连接级失败。** 一条通道打不开
（服务端 `MaxSessions` 满了）与整条连接断了是两件事，
上层的重连策略只该对后者生效。把它们放进同一条继承链，
调用方 `catch (SshConnectionClosedException)` 就会把前者也吞进去。

〔决策〕**`SshPublicKeyException` 与 `SshKeyExchangeException` 都在 `SshException` 之下。**
两者都曾经直接派生自 `Exception`：调用方用 `catch (SshException)` 兜库的失败时漏掉它们，宿主的异常翻译也认不出；
`SshKeyExchangeException` 在建连时还会原样漏给调用方。
〔决策〕**`SshKeyExchangeException` 派生自 `SshConnectException`**，与 `SshNegotiationException` 同一层：它是握手阶段的失败，
而 getting-started 说握手阶段的失败是 `SshConnectException`。曾经直接派生自 `SshException`，按那个类型分流的调用方（宿主的异常翻译）
把它落进兜底分支，建连失败被当成一般错误。重协商时失败也是它（会话随之断开），与重协商时协商失败一样。
`SshPublicKeyException` 的 `Phase` 记 `None`（见上一段）；它曾经记 `KeyExchange`，读本地 `.pub` 时那个阶段并不成立。
`SshKeyExchangeException` 的 `Reason` 记 `ProtocolError`：它报的几乎总是对端给的公开值不合法（长度不对、不在曲线上、弱值）；
「算法没实现」那一类在连接之前就被挡住了（`SshConnection.ConnectAsync` 先核对算法清单）。
清单与实现表万一对不上，那是库自己的编程错误，报 `InvalidOperationException`，不借 `SshException` 的名义。

### 2.1 连接中途断掉：原因先归成公开类型

一条**已经建好**的连接被什么终结了 —— 接收循环、发送泵、保活、重协商里出的任何错 ——
那个原因会被原样交给这条连接上的每一个调用者，以及每一条通道的读者（读会抛出它，而不是像对端发了 EOF 那样读完，
见 `05-connection.md` §4.4）。〔决策〕**交出去之前先归成本库的公开类型：**

| 终结连接的原因 | 调用方拿到的 | `Reason` |
| --- | --- | --- |
| 本来就是 `SshException` | 原样 | 原样 |
| `OperationCanceledException`、`ObjectDisposedException` | 原样 —— 那是本端在收工（取消、释放），不是故障 | — |
| 对端在一个报文的中途关闭了连接 | `SshConnectionClosedException` | `ClosedByPeer` |
| `IOException`、`SocketException`（套接字断了、被重置） | `SshConnectionClosedException` | `ClosedByPeer` |
| 帧格式或完整性校验失败、报文解析失败 | `SshProtocolException` | `ProtocolError` |
| 其它任何意外 | `SshConnectionClosedException`，原异常在 `InnerException` 里 | `Unknown` |

新包出来的这几种 `Phase` 一律记为 `Open`，即使故障发生在重协商期间（已知局限）；原样交出的保留自己的 `Phase`（比如重协商超时是 `Rekeying`）。
〔决策〕**密钥交换本身在重协商时失败**（验签失败、协商不上、交换应答格式非法、对端发了 `DISCONNECT`）：原因码不变，`Phase` 改成 `Rekeying`；
协议错误仍是 `SshProtocolException`，其余是 `SshConnectionClosedException`，原来的异常挂在内层（协商失败时双方的名单还在它上面）。
交换器不分首次与重协商，按首次的口径报 `SshConnectException` / `KeyExchange` —— 曾经原样交出去，按类型分流的调用方会把一条早已建好的连接当成「没连上」。

理由：这个原因会落到使用者的 `catch` 与重连策略上，而那两者都按 `SshException` 与它的 `Reason` 分流
（§3：自动重连只该对「断了」生效）。曾经原样抛出：内部的解析异常类型，使用者按类型 `catch` 不到；
裸的套接字异常绕开了 `Reason`，重连策略分不出它是「断了」；宿主的异常翻译也认不出它们。

〔决策〕**连接自己交出结束原因**：`SshConnection.Completion`（`Task<SshException>`）在连接结束时**以结果**完成 ——
断线时是上表归一之后的那个原因（保活超时、对端关闭、收到 `DISCONNECT` 带着原因码与原话、协议错误……），
本端释放时是 `Aborted` 的 `SshConnectionClosedException`；`CloseReason` 是它的同步版本（还没结束时为 `null`）。
从不以异常完成，没人等它也不会变成未观察的任务异常。`Disconnected` 令牌只说「到此为止」，装不下原因 ——
曾经使用者挂它只能一律报「对端关闭」（宿主的隧道面板连用户自己断开会话也这么报），或者从读管道的结束方式去反推。

**建连期间**（拨通之后，到认证结束）用的是同一套口径，只是 `Phase` 记失败发生的那一步：

| 建连期间出了什么事 | 调用方拿到的 | `Reason` |
| --- | --- | --- |
| 版本交换、密钥交换、认证期间流上的 `IOException` / `SocketException` | `SshConnectionClosedException`，原异常在 `InnerException` 里 | `ClosedByPeer` |
| 对端在一个报文的中途关闭（密钥交换、认证期间） | `SshConnectionClosedException` | `ClosedByPeer` |
| 对端在版本交换期间关闭 | `SshConnectException` | `ClosedByPeer` |
| 帧格式或完整性校验失败（密钥交换、认证期间） | `SshProtocolException` | `ProtocolError` |
| 密钥交换的计算失败（对端公开值不合法） | `SshKeyExchangeException` | `ProtocolError` |
| 调用方回调（主机密钥策略、`IHostKeyTypePreference`、横幅处理器）自己抛的异常 | 原样交还 —— 那不是本库的失败，不归成断开 | — |
| 回调自己抛的取消（调用方的令牌与本库的计时器都没有触发：使用者在询问框上点了「取消」） | `SshConnectException`；认证期间先发 `DISCONNECT(AUTH_CANCELLED_BY_USER)` | `Aborted` |

同一个「断了」按在哪一步、由谁察觉，可能是 `SshConnectException` 也可能是 `SshConnectionClosedException`，
`Reason` 却总是 `ClosedByPeer` —— 判断「是不是断了」看 `Reason`，不看类型。
〔决策〕曾经密钥交换期间的报文中途断开报成 `ProtocolError`，调用方会以为不值得重连；拨通之后的套接字异常原样漏出。现在与会话期间一致。
〔决策〕曾经回调里的 IO 错（宿主的信任库出错、写不了它自己的文件）也被上面第一行改写成可重试的 `ClosedByPeer` —— 重试只会再失败一次；
本库自己读 `known_hosts` 时的 `UnauthorizedAccessException` 则原样漏出。现在回调的异常原样交还，本库读写 `known_hosts` 失败报 `HostKeyStoreFailed`（§3）。
〔决策〕**判超时只看本库自己的那把计时器**（连接计时器、认证计时器、主机密钥裁决的计时器）。曾经调用方没取消的取消一律报成超时：
用户在动态码框上点「取消」得到「认证超时（限 120 秒）」，不限时的主机密钥裁决报「裁决超时（-00:00:00.001）」，
宿主只好在回调里另记一笔、失败之后再认回来。调用方自己取消的照旧原样当取消交还。

拨号阶段不在此列：各个拨号器自己把失败归成带原因的 `SshConnectException`（§3 的 `DnsFailure`、`TcpRefused`、`ProxyRefused`……，见 `09-dialing.md`）。
建连期间的超时、协商失败与主机密钥被拒也由各步自己报（§3）。

---

## 三 `SshFailureReason`

| 取值 | 含义 | 可重试 | 典型下一步 |
| --- | --- | :-: | --- |
| `DnsFailure` | 主机名解析不了 | ✔ | 检查主机名/DNS |
| `TcpRefused` | 连接被拒 | ✔ | 检查端口/服务是否在跑 |
| `TcpTimeout` | 连接超时 | ✔ | 检查防火墙/网络 |
| `TcpUnreachable` | 网络不可达 | ✔ | |
| `ProxyUnreachable` | 连不上代理本身（具体原因在 `InnerException` 与 `Hops`，`09-dialing.md` §2.2） | ✔ | 检查代理地址、端口与它是否在跑 |
| `ProxyRefused` | 代理拒绝转发 | ✔ | **见 §5.2** |
| `ProxyAuthRequired` | 代理要求认证，而没配凭据 | ✘ | 配置代理凭据 |
| `ProxyAuthFailed` | 代理拒绝了配置的凭据 | ✘ | 改对代理的用户名或口令 |
| `TlsFailed` | TLS 握手失败（`DialerChain.Tls`）：证书不可信、名字对不上，或对端说的不是 TLS（`09-dialing.md` §4.4） | ✘ | 见 `Message`；自签证书要钉指纹 |
| `NotAnSshServer` | 对端不说 SSH | ✘ | 端口连错了 |
| `VersionMismatch` | 协议版本不是 2.0 | ✘ | |
| `NegotiationFailed` | 算法无交集 | ✘ | **见 §5.1** |
| `HostKeyRejected` | 主机密钥被拒（`SshConnectException`，`Phase` 为 `KeyExchange`）：策略拒绝（`SshHostKeyVerdict.Reject`）—— 没见过而不许问、使用者拒绝、被 `@revoked`、指纹不在白名单；`K_S` 解析不了、签名验不过、RSA 太短、与协商出的算法对不上（含协商出证书算法而 `K_S` 不是证书，或反过来）；有 CA 担保的主机证书不合格（`03-key-exchange.md` §5.5） | ✘ | 看 `Message`：策略给的原因（`SshHostKeyVerdict.Message`）原样放在里面，附指纹与 `known_hosts` 行号 |
| `HostKeyChanged` | 主机密钥**变了**，两种处境：① 首次交换时策略用 `SshHostKeyVerdict.RejectChanged` 拒绝 —— `KnownHostsPolicy` 在记着的密钥变了、或只记着别的类型时这么报（`SshConnectException`，`Phase` 为 `KeyExchange`，消息里有新旧指纹与行号）；② 重协商时对端出示的主机密钥与首次交换时钉住的不同（`03-key-exchange.md` §8.4），连接以 `SshConnectionClosedException`（`Phase` 为 `Rekeying`）断开 | ✘ | 可能是中间人：不要自动重连，也不要给「信任并记住」的捷径 —— 服务器确实重装了的话，让人去 `known_hosts` 删掉旧的那一行 |
| `HostKeyStoreFailed` | 主机密钥的记录读不出来或写不进去：`known_hosts` 没有权限、被别的进程占着、磁盘满（`SshConnectException`，`Phase` 为 `KeyExchange`）。读不出来时连接不放行；「信任并记住」时写不进去**不**让连接失败，原因记在 `SshConnection.HostKeyPersistFailure`（`03-key-exchange.md` §5.4） | ✘ | 看 `Message`：里面有文件路径与 IO 错误 |

| `AuthenticationMethodExhausted` | 所有方法试完 | ✘ | **见 §5.3** |
| `TwoFactorRequired` | 服务端要 keyboard-interactive 而我们没配 | ✘ | 提示「这台机器需要动态码」 |
| `PasswordExpired` | 服务端要求先改密码，没改成（没配取新密码的回调、这次不改、服务端没改成或问够了次数，04 §5.1） | ✘ | |
| `KeyFileUnreadable` | 私钥 / 证书 / 公钥文件读不出来（不存在、没有权限、IO 错误） | ✘ | 消息里有路径 |
| `KeyFormatInvalid` | 私钥 / 证书 / 公钥的内容格式不对（损坏、截断、参数不成立） | ✘ | |
| `KeyPassphraseRequired` | 加密的私钥需要口令，而没有给 | ✘ | 弹口令输入框（`SshPrivateKeyException.NeedsPassphrase`） |
| `KeyPassphraseIncorrect` | 给了口令，但解不开这把私钥 | ✘ | 再问一次口令 |
| `KeyMismatch` | 凭据材料对不上：证书里的公钥与私钥不是一对、拿主机证书去登录 | ✘ | 证书要与签发时用的那把私钥一起用 |
| `AgentNotRunning` | 本机没在跑 ssh-agent：`SSH_AUTH_SOCK` 没设、套接字不存在或没人监听、Windows 的命名管道在 3 秒内没出现（`07-forwarding.md` §7.1）。agent 转发没开成时 `SshForwardException` 沿用这个原因码 | ✘ | 把 agent 起来（Windows：`Start-Service ssh-agent`） |
| `AgentUnavailable` | 连不上 ssh-agent 的其余情形：端点不可信（命名管道属主不对）、通信中途出错 | ✘ | 看 `Message` |
| `AgentRefused` | ssh-agent 拒绝了请求（`ssh-add -c` 的确认被拒、agent 被锁、钥已不在） | ✘ | |
| `Timeout` | 某阶段超时 | ✔ | |
| `KeepAliveTimeout` | 保活判死 | ✔ | **自动重连只该对这一类生效** |
| `ClosedByPeer` | 对端主动关闭 | ✔ | |
| `Disconnected` | 收到 `SSH_MSG_DISCONNECT` | ✘ 〔未实现：本想按 `DisconnectReason` 细分，目前一律不可重试〕 | 原因码在 `SshConnectionClosedException.DisconnectReason`，对端原话在 `PeerDescription` |
| `ProtocolError` | 对端违反协议 | ✘ | |
| `ChannelOpenFailed` | 通道打不开（`CHANNEL_OPEN_FAILURE`，原因码在 `OpenFailureReason`） | ✘ 〔未实现：本想按原因码细分，目前一律不可重试〕 | |
| `ChannelRequestRejected` | 通道开成了，但通道上的 `exec` / `pty-req` / `shell` / `subsystem` 被拒 | ✘ | 看服务端的 `ForceCommand`、`PermitTTY`、`Subsystem` 配置 |
| `ForwardRejected` | 服务端不接受转发请求（`AllowTcpForwarding no`、`AllowAgentForwarding no` 之类），或本端拒绝了对不上任何转发的入站通道 | ✘ | |
| `ForwardBindFailed` | 本机的监听端口开不了（被占用、没有权限） | ✘ | 换一个端口 |
| `ForwardSetupFailed` | 本机一侧准备转发失败：拿不到 X 显示、`xauth` 跑不起来或失败 | ✘ | |
| `LimitExceeded` | 本端的某个上限到了（并发通道数、会话接收窗口总预算、转发连接数、agent / X11 通道数） | ✘ | |
| `CommandFailed` | 远端命令没有以退出码 0 结束（`SshCommandResult.EnsureSuccess` 抛的 `SshCommandFailedException`） | ✘ | 看 `Result`：stderr、退出码或信号 |
| `InvalidConfiguration` | 配置本身不成立：`ProxyJump` 成环、跳数超限、`ProxyCommand` 模板非法 | ✘ | 改配置 —— 不改的话下一次还是一样 |
| `Aborted` | 本端中止（Dispose / 取消）；建连时回调自己抛的取消（使用者在询问框上点了「取消」，§2.1）也是它 | ✘ | |
| `Unsupported` | 请求的能力（算法、密钥类型、格式版本）对端或本库不支持 | ✘ | |
| `Unknown` | 未分类：连接因意外错误中断（§2.1） | ✘ | 看 `InnerException` |

〔决策〕**`IsRetryable` 是库给的建议，不是承诺。** 它表达的是
「这个失败是否可能因为重试而消失」，不表达「应该重试」——
重试策略是使用者的事（他们才知道用户在等还是在跑批）。

---

## 四 `SshPhase`

```
Dialing → VersionExchange → KeyExchange → Authenticating → Open → Closing
                                 ↑                          │
                                 └──── Rekeying ←───────────┘
```

| 取值 | 说明 |
| --- | --- |
| `Dialing` | DNS、TCP、代理握手、跳板链 |
| `VersionExchange` | 标识串交换 |
| `KeyExchange` | 首次 KEX（含主机密钥验证） |
| `Authenticating` | 认证 |
| `Open` | 正常运行 |
| `Rekeying` | 重协商中 |
| `Closing` | 收尾 |

**失败必须带 Phase。** 同一个 `Timeout` 出现在 `Dialing` 与 `Authenticating`
是完全不同的两件事：前者是网络问题，后者很可能是用户在找手机上的动态码。

---

## 五 三个带结构化上下文的异常

### 5.1 `SshNegotiationException`

```
NegotiationCategory   Category       // KeyExchange / HostKey / EncryptionC2S /
                                     // EncryptionS2C / MacC2S / MacS2C / CompressionC2S / CompressionS2C
IReadOnlyList<string> OfferedByPeer  // 对端 KEXINIT 的该类名单，原样
IReadOnlyList<string> OfferedByUs    // 我们发出去的该类名单，原样
string                PeerVersion    // "SSH-2.0-OpenSSH_9.6p1 ..."
```

**这一条直接替代了「协商失败后再开一条 TCP 明文探对端 KEXINIT」的做法。**
那份名单我们在握手时本来就收到了，只是以前没交出来。

〔决策〕**`Message` 里给一段可读的差异说明**，但**名单必须同时以结构化形式提供** ——
UI 要按类别分行排版、要高亮交集为空的那一类，靠解析 `Message` 做不到。

〔决策〕**展示时过滤掉 `*-cert-v01@openssh.com` 变体**（在使用者一侧做，库照常给全量）。
它们只在对端出示证书时才用得上，列出来会把六个可用算法淹没在十二行里。

### 5.2 `SshConnectException` 的代理上下文

```
IReadOnlyList<SshHopInfo> Hops   // 跳板链/代理链上每一跳：类型、地址、是否成功、耗时
```

〔决策〕失败在链上哪一跳**必须能看出来**。
「连不上 10.0.0.9」和「连不上跳板机 jump.example.com」对用户是两个完全不同的问题，
而现在它们长得一模一样。

`SshHopInfo` 至少有 `Kind`（`Tcp` / `Socks5` / `HttpConnect` / `SshJump` / `ProxyCommand`）、
`Target`、`Succeeded`、`Elapsed`、`Detail`。
填写规则（从近到远、内层原样保留）见 `09-dialing.md` §2.2。

〔决策〕**`FailureKind` 说清是代理与跳板的哪一步**：代理连不上、握手超时、握手中途断开、主机名放不进请求、SOCKS5 凭据超长、SOCKS5 的版本字节 / 方法 / 地址类型 / 应答码（`ProxyReplyCode`）、
HTTP 应答不是 HTTP / 头太大 / 状态码（`ProxyReplyCode` 与状态行 `ProxyReply`）、ProxyCommand 连上之前退出（`ProxyReplyCode` 退出码、`ProxyReply` stderr 末尾）；
跳板连不上、建连超时、不肯转发、转发时出问题、转发超时（`Via` 是跳板、`Target` 是目标），逐个种类见 `09-dialing.md` §2.2、§三、§四、§五、§六。
`DialHops` 改写一个异常只为补上跳的信息时，原来的种类与参数原样带过去。使用者按种类写文案，不解析句子。

〔决策〕HTTP CONNECT 代理拒绝 22 端口时，**在消息里直接给出建议**
（「代理可能只放行 80/443，请改用 SOCKS5 或直连」）。
这是一个高频且用户完全猜不到的失败。

### 5.3 `SshAuthenticationException`

```
IReadOnlyList<AuthAttempt> Attempts      // 见 04-authentication.md §3.4
IReadOnlyList<string>      ServerOffered // 服务端最后给出的可用方法
bool                       PartialSuccessAchieved
```

每条尝试记录带 `Details`（种类加参数，见 [04 §3.4](04-authentication.md)），界面按它用自己的语言写逐条说明，不解析 `Detail`。

`AuthAttempt.Outcome` 里的 `SkippedNotOffered` / `SkippedNoMaterial` 是关键：
没有它们，「私钥文件读不出来」和「服务端不接受公钥认证」在 UI 上是同一句
「用户名或密码不正确」。

---

## 六 `SSH_MSG_DISCONNECT`

| # | 类型 | 字段 |
| :-: | --- | --- |
| 1 | `byte` | 1 |
| 2 | `uint32` | 原因码 |
| 3 | `string` | 描述（UTF-8） |
| 4 | `string` | 语言标记 |

| 码 | 名称 |
| :-: | --- |
| 1 | `HOST_NOT_ALLOWED_TO_CONNECT` |
| 2 | `PROTOCOL_ERROR` |
| 3 | `KEY_EXCHANGE_FAILED` |
| 4 | `RESERVED` |
| 5 | `MAC_ERROR` |
| 6 | `COMPRESSION_ERROR` |
| 7 | `SERVICE_NOT_AVAILABLE` |
| 8 | `PROTOCOL_VERSION_NOT_SUPPORTED` |
| 9 | `HOST_KEY_NOT_VERIFIABLE` |
| 10 | `CONNECTION_LOST` |
| 11 | `BY_APPLICATION` |
| 12 | `TOO_MANY_CONNECTIONS` |
| 13 | `AUTH_CANCELLED_BY_USER` |
| 14 | `NO_MORE_AUTH_METHODS_AVAILABLE` |
| 15 | `ILLEGAL_USER_NAME` |

〔决策〕**服务端给的描述文本必须原样保留**（`SshDisconnectException.PeerDescription`）。
它常常是唯一有用的信息（`"Too many authentication failures"`、
`"No supported authentication methods available"`）。
同时它是**不可信文本**，文档里要写明展示时按不可信内容处理。

〔决策〕**我们断开时也发 `DISCONNECT`**，尽力而为。
不发会让服务端日志里只看到一个 TCP reset，管理员无从判断是网络问题还是客户端主动。
发不发、发哪个码：

| 什么时候 | 原因码 |
| --- | --- |
| 正常释放连接（先限时冲刷已入队的帧，`DISCONNECT` 排在最后，它出去了前面的帧也就出去了） | `BY_APPLICATION`（11） |
| 对端发来的东西解不开、违反了协议（握手、认证、会话期间都算；帧层的完整性校验失败也报这个，不单独报 `MAC_ERROR` —— 把校验细节回送给对端是侧信道；密钥交换里的公开值不合格另算，见下面第二行） | `PROTOCOL_ERROR`（2） |
| 算法协商不上 | `KEY_EXCHANGE_FAILED`（3） |
| 密钥交换里对端的公开值不合格（长度、编码、不在曲线上、X25519 全零、DH 越界、GEX 的 `p` 非素数与 `g` 越界、混合方法的总长；[03 §3.7.4](03-key-exchange.md)）。本端的原因仍是 `ProtocolError` | `KEY_EXCHANGE_FAILED`（3） |
| 主机密钥被拒、验不过，或重协商时换了 | `HOST_KEY_NOT_VERIFIABLE`（9） |
| 认证方法用尽（含要动态码而没配、要先改密码） | `NO_MORE_AUTH_METHODS_AVAILABLE`（14） |
| 使用者在询问框上取消了认证（§2.1） | `AUTH_CANCELLED_BY_USER`（13） |
| 连接已经断了（套接字错、对端在报文中途走了）、超时、本端配置问题 | 不发 —— 前两种发了也没人收，后几种不是对端的事 |

版本交换完成之前不发（那时对端还不一定说 SSH）。每次最多等 2 秒，发不出去不报。

---

## 七 度量（`System.Diagnostics.Metrics`）

Meter 名：`VelaShell.Ssh`（`SshMetrics.MeterName`；转发的那一组另有自己的 Meter `VelaShell.Ssh.Forwarding`，见 [`07-forwarding.md`](07-forwarding.md) §5）。
对外只交出名字，仪表本身不公开 —— 交出可写的计数器等于让任何人都能往里记账；没有订阅者时每次记账只是一次判断。

| 仪表 | 类型 | 标签 | 何时记 |
| --- | --- | --- | --- |
| `velashell.ssh.connections.active` | UpDownCounter | `host` | 认证完成 +1，连接结束（对端断开、出错、本端释放）-1 |
| `velashell.ssh.connect.duration` | Histogram (ms) | `host`、`outcome`、`phase` | 每次 `ConnectAsync` 一笔：从拨号到认证完成或失败。`outcome` 是 `Success`、`SshFailureReason` 的名字、`Canceled`（调用方取消）或 `CallbackFailed`（使用者的回调自己抛了）；`phase` 是停在哪一步（成功为 `Open`） |
| `velashell.ssh.bytes` | Counter | `host`、`direction` | 流上实际收发的全部字节（版本标识串、报文头、填充、MAC，压缩之后的），与 `SshConnection.BytesSent` / `BytesReceived` 一致 |
| `velashell.ssh.packets` | Counter | `host`、`direction` | 每个报文一次，与 `SshConnection.PacketsSent` / `PacketsReceived` 一致 |
| `velashell.ssh.rekeys` | Counter | `host` | 每完成一次重协商（与 `SshConnection.RekeyCount` 一致） |
| `velashell.ssh.channels.active` | UpDownCounter | `host`、`type` | 对端确认（或我们接受对端开的）+1，通道收尾 -1 |
| `velashell.ssh.channel.window` | Histogram (bytes) | `host`、`type` | 自适应窗口的实际取值：开通道时记一次，每扩、缩一次再记一次 |
| `velashell.ssh.sftp.inflight` | Histogram | `host` | 管线深度的实际取值：每发一个请求记一次此刻的在途数（含它自己） |
| `velashell.ssh.forward.*` | 见 [`07-forwarding.md`](07-forwarding.md) §5 | | |

`direction` 取 `sent` / `received`（与转发的那一组一致）。`type` 只取已知的通道类型（`session`、`direct-tcpip`、`forwarded-tcpip`、`x11`、
`auth-agent@openssh.com`、`direct-streamlocal@openssh.com`、`forwarded-streamlocal@openssh.com`），别的一律记 `other` ——
对端开过来的通道类型是对端给的字符串，照单全收就是让对端决定标签基数。

单个连接、单条通道的实时数字不必订阅度量，直接读对象上的属性：`SshConnection.BytesSent` / `BytesReceived` / `PacketsSent` / `PacketsReceived`
（线上），`SshChannel.BytesSent` / `BytesReceived`（应用数据：`CHANNEL_DATA` 与 `CHANNEL_EXTENDED_DATA` 的数据段，不含协议开销；收到的读没读走都算）。

〔决策〕**`host` 标签用「逻辑目标」而不是 IP。** 跳板链上的最终目标才是用户认识的东西：经跳板时，各跳的连接（含跳板那一跳的
`connect.duration`、`bytes`）也记在最终目标的名下。测试里不经 `ConnectAsync` 直接建的连接不记度量。

〔决策〕**不给密钥指纹、用户名之类打标签。** 标签会进时序数据库，基数爆炸是一方面，
把用户名写进可被广泛查询的指标里是另一方面。

---

## 八 追踪（`ActivitySource`）

〔现状〕**还没有实现**，下表是规划。

Source 名：`VelaShell.Ssh`

| Activity | 何时 |
| --- | --- |
| `ssh.connect` | 整个建连，子 span 为各 Phase 与各跳 |
| `ssh.command` | 一次 `RunAsync` |
| `ssh.sftp.<op>` | 单个 SFTP 操作（只在采样命中时建，见下） |

〔决策〕**SFTP 的单操作 span 默认不建。** 一次目录传输会产生上万个操作，
逐个建 Activity 的开销本身就会改变性能特征。默认只在
`SftpFileSystem` 级别建一个聚合 span；需要细粒度时显式打开。

---

## 九 报文旁路 `IPacketTap`

```
interface IPacketTap
{
    void OnPacket(in PacketTapRecord record);
}

readonly ref struct PacketTapRecord
{
    PacketDirection Direction;     // Inbound / Outbound
    byte            MessageNumber;
    int             Length;        // 载荷长度（含消息编号）
    uint            SequenceNumber;
    uint?           ChannelNumber; // 通道消息（91–100）才有
    ReadOnlySpan<byte> Payload;    // **默认为空**，见下
}
```

挂法：`SshConnectionOptions.PacketTap`。跳板链上每一跳是各自的连接，要看哪一跳就在那一跳的连接参数里设。

**四条硬规则**：

1. **默认不启用。** `PacketTap` 为 `null`（默认）时收发路径上只多一次判空。
2. **载荷默认不给。** 要给必须显式设 `SshConnectionOptions.AllowPacketTapPayload = true`，
   且该选项的文档里**必须**写明它会带出密码、密钥与文件内容。
3. **认证报文的载荷永远不给**，即使 `AllowPacketTapPayload = true`：消息编号落在 50–79
   （[RFC 4252](https://www.rfc-editor.org/rfc/rfc4252) 给用户认证协议留的段，含键盘交互的提问与回答）的报文，
   旁路只看得到元信息。
   〔决策〕这一条不提供开关 —— 没有任何排错场景值得把密码打进日志，
   而提供了开关就一定会有人在生产上打开它。
4. **敏感交互租约存续时，整条连接双向载荷都不给**，即使已经打开 `AllowPacketTapPayload`。
   SSH 登录之后的口令（例如受控 sudo 认证）走的是普通通道数据，不属于 50–79 的保护范围；规则 3 不能代替这一条。

### 9.1 敏感交互的连接级载荷抑制租约

〔规格先行；下游专用契约：**SDK 2.1.0**〕下面定义行为要求。SSH 库不单独发包；下游 sudo 已在 Ubuntu/Debian 的真实 SSH/PTY/PAM 上验证执行内核，不能据此扩大为完整 GUI 与插件 IPC 的真实端到端验收。

调用方在同一条 SSH 连接上取得一个**载荷抑制租约**，必须在可能产生敏感交互的通道请求、提示、输入或输出之前取得成功。
租约只收紧诊断可见性，不改变 SSH 报文、通道内容、远端执行、应用正常读取输出的行为，也不替调用方决定命令授权或认证。

- **连接范围、引用计数。** 每个租约属于取得它的那条连接；只要还有一个有效租约，该连接的所有通道、两个方向以及重协商报文的 `Payload` 都为空。
  多个敏感执行可以交叠，先结束的执行不得解除后一个执行的保护。重复释放同一个租约没有副作用，不得重复减数或造成计数下溢。
  不允许用一个可被并发操作互相覆盖的布尔开关代替租约；其它连接不受影响，跳板链仍按每跳独立连接处理。
- **只保留元数据。** 旁路仍得到方向、消息编号、原始载荷长度、序号，以及原本就能取得的通道号；流量和报文计数照常累计。
  不能以摘要、前缀、编码文本或附加字段重新带出载荷。认证提示、一次性标记、口令及可能的对端回显同样被抑制；不能只遮掉发送的密码行。
- **覆盖异步收尾。** 调用方持有租约贯穿敏感通道的创建、认证、执行、取消和异步关闭；不能在写密码返回、认证成功、发出取消请求或收到进程退出状态时就提前释放。
  只有相关通道的在途发送、接收及关闭处理已收尾，不会再向旁路交出该执行的载荷，或整条连接已终止诊断交付时，才可结束保护。
  失败、超时、用户拒绝、断线与释放均须走同样的收尾路径；不能因抛异常而泄漏租约，也不能因过早清零而暴露迟到的数据。
- **并发安全、边界明确。** 取得、释放、双向旁路回调与连接释放可以并发；成功取得租约之后开始的旁路交付必须看到抑制状态。
  取得租约之前已经交给旁路的载荷无法追溯撤回，因此必须先取得租约再启动敏感工作。租约关闭与连接关闭的竞态不得恢复一条仍有有效租约的连接的载荷，
  不得引发死锁，也不得影响原有“旁路异常不影响连接”的规则。连接已关闭时不能再成功开始敏感工作，遗留租约的释放仍安全、幂等。
- **不改用户设置。** 租约不修改或重新保存 `PacketTap` / `AllowPacketTapPayload`，也不在结束时盲目写回一个旧值。
  最后一个租约结束后，非认证报文按连接原有配置决定是否给载荷；认证报文永远仍只给元数据。原本没有开启载荷的连接不会因租约结束而开启。

〔验收边界〕在显式开启载荷旁路、同时运行多个通道的条件下，验证双向抑制、重叠租约、重复释放、取消/失败的迟到数据与连接释放竞态；
验证元数据保留、其它连接不受影响以及最后一个租约结束后仅恢复原配置允许的可见性。
〔实现收尾〕敏感执行的本端 Dispose 返回不代表对端已经停止交付；等待对端 CLOSE 与发送泵结束，或整条连接终止诊断后才释放租约。
如果开通道失败而没有可跟踪的通道，或排空失败，保守持有至连接结束；这可能延长同连接其它通道的载荷抑制，是保密优先的明确取舍。
租约只保护库内诊断旁路，不是对应用输出的自动脱敏器：调用方仍须将敏感认证阶段内容与正常命令输出分开，不把口令、认证标记或原始认证内容交给插件、模型、IPC、历史或日志。
这不提供任意终端口令检测或凭据自动填充，也不改变现有用户终端的凭据交互决定。

〔实现〕挂在帧层收发的两个点上：收到的在解密、解压之后，发出的在压缩、加密之前 —— 旁路看到的是明文载荷层，
序号是该方向上这个报文的序号（严格 KEX 下每次 `NEWKEYS` 归零，见 [`03-key-exchange.md`](03-key-exchange.md) §6）。
`Payload` 借的是传输的缓冲，只在这次 `OnPacket` 调用期间有效（所以记录是 `ref struct`），要留就复制。
回调跑在收发循环上，必须很快、不能阻塞；它抛的异常被吞掉 —— 旁路是使用者的代码，它出错不该让连接断开。

用途：连接诊断面板、协议级排错、录制回放。

---

## 十 日志

用 `ILogger`，但**日志不是 API**：任何使用者需要程序化消费的东西
都必须同时以结构化形式出现在异常、事件或度量里。

〔现状〕**库里还没有日志器**：`SshConnectionOptions` 上没有 `ILoggerFactory`，下表是规划。
规格各处写的「记一条 debug 日志」目前都只做了动作本身（丢弃、忽略），没有记录；
「对端不支持严格 KEX」可以从 `SshConnection.Algorithms.StrictKeyExchange` 读到，但没有 Warning。

| 级别 | 内容 |
| --- | --- |
| `Trace` | 每个报文的元信息（等价于 `IPacketTap` 不含载荷） |
| `Debug` | 状态迁移、协商结果、窗口调整、管线深度变化 |
| `Information` | 连接建立/断开、认证成功、转发启停 |
| `Warning` | 降级（SHA-1 签名、无严格 KEX）、重试、单条转发连接失败 |
| `Error` | 连接级失败 |

〔决策〕**「对端不支持严格 KEX」必须记 Warning。** 它不阻断连接（§03 6），
但使用者应当能看见 —— 这是一条实打实的安全属性缺失。

〔决策〕**热路径的日志用 `LoggerMessage` 源生成器。**
每报文一条 Trace 日志，若走字符串插值会在满速传输时成为主要开销。
