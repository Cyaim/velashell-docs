# 11 · 自动化与 AI

两者都是"插件系统之上的应用层",共享同一原则:**宿主提供骨架与安全
边界,插件提供血肉**。

## 1. 自动化(S5/S6)

### 1.1 模型:规则 = 触发器 + 条件 + 动作

```text
触发器(Trigger)   会话连接成功 / 传输完成 / 定时(cron)/ 手动命令 / 插件自定义触发器
条件(Condition)   when 表达式(复用 03 §6 文法):主机匹配、时间窗、上次结果…
动作(Action)      内置动作(执行远程命令、上传/下载、发通知、写终端…)/ 插件自定义动作
```

规则本体由**宿主**持有(存储、调度、执行编排、日志),插件的参与方式
有两种,刻意分开:

1. **插件消费宿主事件自己干活**(07 §8 的 vela.events + 各能力):
   适合复杂逻辑,一切行为计入该插件的权限与审计。
2. **插件向规则引擎贡献触发器/动作**(manifest `contributes.automation`):
   用户在宿主的"自动化"页面里可视化编排,把插件能力当积木用。执行时
   宿主回调插件(`automation/evaluateTrigger`、`automation/runAction`),
   动作运行仍受该插件自身权限约束——规则引擎不放大权限。

### 1.2 安全边界

- 规则创建/修改只能由**用户在宿主 UI** 完成;插件只能"建议"规则
  (`SuggestRuleAsync` 弹确认卡片),不能静默创建——自动化 + 
  `remote.exec` 的组合杀伤力大,必须人在环。
- 每条规则显示将动用的权限汇总;含高敏动作(remote.exec/terminal.write)
  的规则首次运行前逐条确认,可勾选"此规则以后自动运行"。
- 规则运行历史(时间、触发原因、动作输出摘要、成败)持久化,失败可
  一键重跑;连续失败 N 次自动挂起规则并通知。
- cron 触发的插件唤醒走 `onSchedule` 激活事件,受"错峰 + 最小间隔
  1min"约束,防插件把宿主变成刷子。

### 1.3 内置动作(宿主自带,无需插件)

执行远程命令(exec 通道)、上传/下载文件、发通知、打开会话、写终端
(高敏)、运行本地程序(高敏,显式路径)。这让轻量自动化零插件可用,
插件负责长尾。

### 1.4 AI 插件的托管执行与恢复（SDK 2.1.0）

本节记录 AI 插件当前新增源码，与上方规则引擎、下方宿主 AI 网关的远期设计分开；不是把网关/通用权限系统宣告为已实现。

**托管命令**仅在应用运行期间保存，不做断线/重启恢复：

| 工具 | 参数与行为 |
| --- | --- |
| `run_command` | `timeoutSeconds=0` 默认无执行死线（显式值上限 86400）；`waitSeconds=2`、最大 10，仅短等窗口，到期返回 running/taskId，不杀命令；`monitoring=false`；`elevated=false` |
| `run_on_sessions` | 最多 16 个目标排入共享 4 流槽位，未开流项取消后不再启动，不直接撞宿主并发上限 |
| `task_status` | `taskId`、`cursor=0`，返回状态及增量输出 |
| `task_wait` | `taskId`、`waitSeconds=2`、最大 30、`cursor=0`；等完不代表执行死线，也不取消命令 |
| `task_cancel` | 请求取消本轮任务；清理不会因读取 scope 已撤销而被阻挡 |

状态为 Starting / AwaitingUser / Running / Succeeded / Failed / Cancelled / TimedOut / Unknown。普通结果保留 stdout、stderr、退出码、超时/截断信息；sudo 是 `combined`，保留原始换行/CR 分片而不伪造 stderr。运行后取消无法确认远端停止时为 Unknown，`cancellationRequested=true`、`remoteStopConfirmed=false`；未开流取消为 Cancelled 且已确认未启动。SSH 断线、sudo Exited 缺退出码等为 Unknown；已启动 TimedOut 且无停止确认也阻断本 run 后续自动工具副作用，绝不由框架重跑。认证取消/失败/不支持明确表示用户命令未启动，不自动降级普通执行。

每 run 最多 128 个任务；输出是线程安全有界队列（32K 字符、256 条、每条最多 4096 字符），用 `NextCursor` / `EarliestCursor` / `OutputTruncated` 显示过期与截断。taskId 只在创建它的 run scope 可读，绑定入口/对话/固定 sessionId，不跨聊天或 MCP 客户端。启动、查询、完成重验会话 scope；当前缓存与撤权边界见 [安全 §1.1](12-security-threat-model.md#11-此次-sudo-与-ai-托管执行的实际边界sdk-210)。

命令必须以前台进程留在托管通道。不会解析/改写任意 shell 的 `&`、nohup、daemonize，自行脱离的进程不受监控，父进程退出不能证明它完成。显式 `monitoring=true` 的无限日志流仅本地等退出或 Stop，不自动唤醒/轮询模型。

**提权是两层明确意图。** `elevated=true` 传不含 sudo 包装的实际命令，整条 root 范围包含管道和重定向；插件 Ask/ReadOnlyAuto 独立审批、不记首词批准，Bypass 只跳插件卡片，不能跳过宿主逐次可信确认（NOPASSWD 同样确认）。MCP 无 Handler 时 Ask/ReadOnlyAuto 拒绝，Bypass 可请求桌面确认。密码不经过插件或模型；不是用户终端自动填充。详见 [能力 API §3](07-capability-apis.md#3-velaremoteexec--远程执行s5s6)。

**拒绝是运行级停止边界。** 插件审批返回 false/缺少 Handler，或宿主返回 AuthenticationCancelled（含拒绝 root 确认/取消密码），都会标记当前 run 已拒绝；模型请求前、工具调用前及后台终态续接检查后以 `ExecutionDenied` 强制暂停，桌面/Bridge 显示本地化暂停原因，不依赖模型遵守“不要重试”字符串。桌面/Bridge 新用户轮次会创建新 run，不能在已拒绝的原 run 内继续副作用。MCP 则在每客户端 McpToolHost 创建固定 scope；拒绝后该 host 后续工具都拒绝，仅 `task_cancel` 保留清理入口。下一条 HTTP/tool call 不会复位，须新建 host/session 或重新连接、刷新授权配置后取得新 scope。

**恢复的是模型步骤，不是整轮副作用。** 桌面聊天/Bridge 每会话串行推进，共享协调器；工具账本按 runId + CallId 复用同 id 同参的在途/完成结果，同 id 异参暂停，不把不同 id 的相同命令一概去重。工具帧只在完整 ToolCalls 步骤、参数完整验证后释放；未完成调用/推理不重放。每完成模型步骤与工具结果配对保留 checkpoint，部分正文以明确本地化续接边界保留，不承诺无法证明的无缝去重。

瞬时网络/429/适当 5xx 每步骤最多恢复 2 次（默认 400/800ms，尊重可取得的 Retry-After；超过 120 秒暂停），每次真实请求经 Steering 复验账号/凭据，取消/拒绝/确定性错误不重试。UI 25 步、Bridge 20 步是显式批次边界；当前固定总预算为 100 模型步骤、200000 token、8 批次，不是用户可配置项，也不是独立计价费用上限。无进展、重复调用或预算耗尽明确暂停，不无限续批。

有限后台任务完成后以本地信号唤醒所属对话，再向模型加入新的终态通知，不为原 CallId 伪造第二个工具结果。本地等待不请求模型、无默认 10 分钟等待死线；Bridge 不扩大现有授权或总超时。Stop、入口 detach/销毁、插件卸载、MCP 会话删除/过期/设置 reload、Bridge 授权 reload 取消旧 run 并异步收尾，废弃迟到唤醒；切换聊天不串入其它会话。MCP 只给任务查询/等待/取消协议，外部模型的自动继续由 MCP 客户端负责。

**发布与验证状态：**

- SDK 2.1.0 已正式发布并由 NuGet.org 恢复，`src/tests` 中央包及 AI 显式引用已升级；AI 清单为 `minSdkVersion: 2.1.0`、`apiLevel: 2`。正式包缺类型的旧阻塞已解除，不使用本地包或临时源。
- 正式 SDK 装配后的执行 RPC、Capability 与 AI sudo 参数化回归已通过。Infrastructure 全量报告中，平台/环境跳过与 `[SKIP]` 早退仍算未验证，不能把报告“通过”全当实际执行。宿主 sudo 对话框及本地化/HeadlessDispatch/DialogButtonStyle 相关门禁已通过，这不是全 UI 回归。最新拒绝守卫修复后的全 AI 与 SSH 定向回归已通过；最终 Debug 严格非增量构建已通过（警告视为错误，启用构建期代码风格检查，零警告/零错误），风格收尾后定向回归也已重新编译通过，不把旧检查点套到新修复。
- Ubuntu 24.04.5 LTS（sudo 1.9.15p5）与 Debian 12 bookworm（sudo 1.9.13p3）在真实 sshd/PTY/PAM 上各通过完整 sudo 矩阵：UTF-8/requiretty、NOPASSWD 仍确认、错密重试与上限、取消、受限 sudoers、假提示/stdin 关闭、断线 Unknown、长前台完成后下一命令、显式 root 与外部取消；同时启用 payload tap 检查秘密不泄漏。前轮 apt 未 ready/策略拒绝误判失败均保留记录，修复后重跑，不再是当前阻塞。
- 真实验证针对执行内核，不等于用户真实点击 GUI → 插件 IPC → 远端的整链实测；内存分片/EOF 状态机与宿主层回归是独立证据。复杂 PAM/OTP 不支持，认证超时与迟到租约的综合动态验收仍不应从现有用例推断。
- 交付前按同一最终源码完成正式包构建、拒绝停止回归与宿主 UI/IPC 检查，保留平台/环境未验范围；全局版本横幅仍由发版脚本管理，不手工改写。

## 2. AI 网关(S3,远期主线)

### 2.1 为什么是"网关"而不是"每个插件自带 SDK 调模型"

- **密钥归宿主**:用户在宿主设置一次模型提供方(Anthropic/OpenAI/
  本地 Ollama…)与 API key,密钥存 OS 凭据库,插件永远拿不到。
- **额度与审计**:按插件统计 token 消耗、限额、审计"哪个插件把什么
  内容发给了模型"——这对运维工具至关重要(终端内容可能含机密)。
- **一次接入,处处可用**:模型供应商演进由宿主消化,插件面对稳定的
  抽象接口。

### 2.2 接口形态(vela.ai,权限 `ai.invoke`)

```csharp
public interface IAi
{
    Task<AiModelInfo[]> ListModelsAsync(CancellationToken ct);       // 用户配置的可用档位(如 fast/balanced/powerful,不暴露密钥)
    IAsyncEnumerable<AiDelta> ChatStreamAsync(AiRequest req, CancellationToken ct);
        // AiRequest: 档位、messages、可选 tools(插件提供的工具回调经 RPC 回插件执行)
    Task<float[][]> EmbedAsync(string model, string[] inputs, CancellationToken ct);
}
```

- 工具调用(function calling)回路:模型请求工具 → 宿主经 RPC 调插件
  的工具实现 → 结果回模型。插件工具的执行仍受其权限约束(AI 不是
  权限旁路)。
- **数据出境提示**:插件首次把 `terminal.read` 获得的内容送入
  ChatStream 时,授权提示明确"终端内容将发送给 <模型提供方>";
  设置里提供"禁止任何插件向 AI 发送终端内容"总开关。
- 宿主侧实现建议直接采用官方 SDK против各供应商;本地模型经 OpenAI
  兼容端点接入,统一在网关内路由。

### 2.3 与自动化的合流

AI 动作("总结这次部署输出并通知我")= 规则引擎的动作调 ai.invoke;
AI 触发器(异常日志语义检测)= 插件订阅终端流 + 网关判定 + 触发规则。
两套骨架拼起来即"智能运维"场景,无需新机制。

## 3. 开发计划(本分项)

自动化(依赖里程碑 M3 后启动):

| 任务 | 说明 | 依赖 | 估算 |
| --- | --- | --- | --- |
| T-1 | 规则模型 + 存储 + 调度器(cron/事件桥接),内置动作第一批 | C-3, C-5 | 5d |
| T-2 | 自动化管理页(规则列表/编辑器/运行历史)| T-1 | 5d |
| T-3 | 插件触发器/动作贡献 + 回调协议 + SuggestRule 确认流 | T-1, M-4 | 3d |
| T-4 | 高敏规则确认流 + 失败挂起 + 审计接线 | T-2, B-6 | 2d |
| T-5 | auto-runner 官方示例(接 S-9)| T-3 | — |

AI 网关(独立轨道,可与 M6+ 并行):

| 任务 | 说明 | 依赖 | 估算 |
| --- | --- | --- | --- |
| I-1 | 网关抽象 + 供应商双实现(Anthropic + OpenAI 兼容端点)+ 密钥管理设置页 | — | 5d |
| I-2 | vela.ai 能力域(流式、工具回路、按插件计量限额)| I-1, C-1 | 4d |
| I-3 | 数据出境护栏(终端内容标记与总开关、审计)| I-2, B-6 | 2d |
| I-4 | AI 助手官方示例插件(S3:解释报错/生成命令回填)| I-2, C-5 | 4d |

验收:S5——"会话连上 prod-* 自动跑巡检脚本并把结果发通知"全程可视化
编排完成;S3——AI 示例插件在授权后能解释终端选中报错,且审计页能看到
发送内容摘要。
