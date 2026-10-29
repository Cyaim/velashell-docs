# RDP 数据交换设计与实施计划

> 状态：阶段计划已批准，第一阶段实施中、尚未验收。本文记录设计边界与验收门禁，不把离线测试结果写成真实远端互操作结论。

## 进度

- [x] 固定 1Remote 对比证据，保存中英文计划并挂入索引。
- [ ] 阶段一 A：剪贴板故障隔离与取消回归。托管实现已合入；严格构建和 163 项离线回归通过；Rust 静态测试 2/2 通过并已验证新 DLL 的故障隔离导出。真实远端互操作未验收，因此不勾选。
- [ ] 阶段一 B：原生扩展构建、ABI 与最终 VPX 门禁。已接 CI/release 测试前构建（固定 Rust 1.94.1）、`VelaRequireRdpNativeExtension` 强制门禁和最终包提取验证；Rust 2/2 测试、Release DLL 构建、新能力导出检查及最终 VPX 163/163 回归已通过。真实 GitHub 工作流未运行，因此仍不勾选。
- [ ] 阶段二：活动会话文本变化通知。
- [ ] 阶段三：Windows 双向文件剪贴板。
- [ ] 阶段四：目录映射与动态设备。

阶段只有在重新构建和对应测试通过后才勾选；真实远端测试另行标记。

**最新门禁状态**：为避免 ABI=1 的旧 DLL 被误判为包含故障隔离，新增 `VelaClipboard_fault_isolation_version() == 1` 发布检查；旧 DLL 已按预期被拒绝。此次严格托管构建零警告/错误，最终离线回归 **163/163 通过、零跳过**，包含最终 VPX 正向验证；Rust 静态测试 **2/2 通过**，Release DLL SHA-256 为 `FD3EA8A24328DC0075CCAC2F37DE22AF64A4071D47D98CBD3A0741EA5BCC5BAF`。未运行 GitHub 工作流或连接真实远端。

## 1. 参考结论

本计划参考 1Remote 固定提交 `f1cfdd1fc518c236e5bd339d3f0b74452f8f9303`。1Remote 使用 Windows `MsTscAx` ActiveX 控件，通过 `RedirectClipboard`、`RedirectDrives`、`DriveCollection` 和动态设备通知配置重定向；剪贴板协商、文件剪贴板和磁盘访问由微软 RDP 控件完成。本仓库继续使用 IronRDP，不嵌入 ActiveX，也不复制 1Remote 的 GPL-3.0 源码。

微软的 RDP 配置把剪贴板与磁盘重定向作为不同开关，但服务端策略禁止磁盘重定向时，也会阻止通过剪贴板传输文件；这不等于协议实现必须共享整块磁盘。文件剪贴板属于 MS-RDPECLIP 的文件描述符、内容请求/响应和文件流机制，不能用 Unicode 文本格式替代。

参考：

- [1Remote 重定向配置](https://github.com/1Remote/1Remote/blob/f1cfdd1fc518c236e5bd339d3f0b74452f8f9303/Ui/View/Host/ProtocolHosts/AxMsRdpClient09Host.xaml.cs#L318-L374)
- [1Remote 生成 RDP 重定向属性](https://github.com/1Remote/1Remote/blob/f1cfdd1fc518c236e5bd339d3f0b74452f8f9303/Ui/Model/Protocol/RDP.cs#L821-L840)
- [微软 RedirectClipboard 属性](https://learn.microsoft.com/en-us/windows/win32/termserv/imsrdpclientadvancedsettings5-redirectclipboard)
- [微软剪贴板重定向策略](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-clipboard)
- [MS-RDPECLIP 文件流](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpeclip/6119fce5-72b8-4ec3-8871-f0b785ededd3)

## 2. 当前实现边界

`VelaShell.Plugin.Rdp` 当前提供：

- 受限的 Unicode 文本 CLIPRDR：文本上限 2 MiB，格式列表和出站队列有界，焦点与会话 epoch 隔离，Ctrl+V 等待公告 ACK。
- Windows x64 静态 RDPDR 逻辑盘映射：默认关闭、默认只读，配置形如 `D:\;E:\`；暴露的是完整逻辑盘根，不是某个文件或目录。

当前不提供：

- 图片、HTML、富文本剪贴板；
- 本地 Explorer 复制文件后在远端粘贴，或反向文件剪贴板；
- 目录级共享、UNC 映射、动态热插拔；
- 真实 Windows 服务端互操作保证。离线状态机、原生 ABI、Headless 和设置测试不能替代真实远端验收。

文件剪贴板与磁盘映射必须保持独立配置。开启文本剪贴板不能隐式授予本地文件读取；开启整卷映射也不能被描述成文件剪贴板。

## 3. 分阶段实施

### 阶段一：故障隔离与发布一致性

先修改 `Ui/RdpClipboardBinding.cs`、`Ui/RdpInputQueue.cs`、`Engine/RdpSession.cs`、`Engine/RdpClipboardState.cs` 和现有错误横幅路径：剪贴板拒绝、超时、超限或本地剪贴板争用只取消当前粘贴或暂停剪贴板，不应因此断开桌面；真实输入发送、socket 写入、不可恢复的 native 状态损坏仍按会话故障处理。

取消必须带操作代次，不能让已入队操作或迟到结果完成下一次粘贴。保留无 wire ID 的请求槽和 ACK 排空规则，不通过原地 Disable/Enable 伪造恢复。失焦、卸载和断线主动取消准备，匹配的 key-up 与 `ReleaseAll` 仍要发送。

同时在 CI/release 的测试之前构建固定 IronRDP 原生扩展，并对最终 VPX 做 ABI、摘要、RID 和许可证检查。`native/artifacts` 是本地产物，不能作为干净 checkout 发布成功的前提。

验收：剪贴板失败不调用 `DisconnectAsync`；本次粘贴不发送残缺按键；后续普通输入正常；真实传输故障仍断线；最终包确实含 win-x64 扩展。

### 阶段二：活动会话的文本变化同步

复用 `RdpClipboardBinding` 的 epoch、合并通知和 `TopLevel.Clipboard`。Windows 优先评估 Avalonia 公开的 `Win32Properties.AddWndProcHookCallback` / `RemoveWndProcHookCallback`，只监听活动桌面，不进行所有标签页的全局轮询。远端回写要带来源/代次，抑制反馈循环；监听更新与 Ctrl+V 的粘贴屏障分离。非 Windows 保留明确的焦点/按键边界。

验收包括外部进程更新、右键粘贴、`Shift+Insert`、两会话切换、通知洪峰、回写反馈、视图销毁和重挂载。

### 阶段三：Windows 文件剪贴板

先扩展 MS-RDPECLIP 文件子状态机，再接 Windows OLE/系统剪贴板适配层。协议需要文件描述符、文件内容范围请求/响应、分块、有界并发、取消、锁和剪贴板代次，而不是仅公告 Unicode 文本。

从单文件双向复制开始，再扩展多文件和目录；首版只复制不删除源文件。路径、名称、重解析点、目录遍历、大小、偏移和文件句柄必须有边界检查。远端不得凭文件索引反查任意本地路径。

文件剪贴板不能简单套用文本“失焦即失效”：用户从远程桌面切到本地 Explorer 后仍需要读取已经发布的文件对象。活动会话停止发布新公告，但已发布对象绑定原会话、剪贴板代次和文件白名单；剪贴板被替换、断线、Dispose 或租约到期时退休对象。COM 回调不得同步阻塞 UI 或网络泵。

新增 `redirectFileClipboard` 默认关闭，与 `redirectClipboard` 和 `redirectDrives` 分开。先验证普通文件，再决定目录和大文件行为；不顺带承诺图片/HTML。

### 阶段四：目录映射与动态设备

先审计固定上游 WindowsRdpdrBackend 的根句柄、相对路径和只读执行，再扩展目录根。现有完整盘根配置保持兼容；新配置使用明确的目录根、共享名和只读属性，不用字符串前缀判断代替句柄隔离。动态设备单独设计设备授权、移除时未决 I/O、重插和断线生命周期。UNC 继续拒绝，除非边界得到独立验证。

## 4. 验证与发布门禁

每阶段必须重新构建后运行 `VelaShell.Plugin.Rdp.Tests`，常规回归排除 `ManualPerformance`，并运行 runtime-async 关闭的严格非增量构建。集合/span 断言中先 await 到局部变量，再调用断言。原生测试覆盖 ABI、扩展存在性和最终 VPX 提取，不以缺少扩展时的“明确拒绝”测试代替交付验证。

真实服务端测试仅在目标与授权明确后执行：双向文本、服务端策略拒绝、RDPDR 实际读写/只读拒绝、文件单/多文件复制、取消、断线和大文件。未执行的项目明确标记为未验收。

实现仓库的插件 README 保留面向开发者的当前事实；本计划的中英文镜像必须同步更新。提交和 PR 另按仓库约定处理，不在本计划阶段自动提交。
