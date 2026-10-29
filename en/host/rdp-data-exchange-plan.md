# RDP Data Exchange Plan

> Status: The phased plan is approved; Phase 1 is being implemented and has not passed acceptance. This document records design boundaries and acceptance gates. Offline test results must not be presented as real remote interoperability.

## Progress

- [x] Pin the 1Remote comparison evidence, save both language versions, and add index links.
- [ ] Phase 1A: clipboard failure isolation and cancellation regressions. The managed implementation is merged; strict rebuild and 163 offline regressions passed; Rust static tests passed 2/2 and the new DLL's fault-isolation export was verified. Real remote interoperability is unverified, so this phase is not checked off.
- [ ] Phase 1B: native extension build, ABI, and final VPX gates. CI/release now builds before managed tests with Rust 1.94.1, enforces `VelaRequireRdpNativeExtension`, and verifies the extracted final package. Rust tests 2/2, the Release DLL build, the new capability export, and the final VPX regression (163/163) passed. GitHub workflows have not run, so this phase is not checked off.
- [ ] Phase 2: active-session text change notifications.
- [ ] Phase 3: bidirectional Windows file clipboard.
- [ ] Phase 4: directory mappings and dynamic devices.

Check off a phase only after rebuilding and passing its tests; record real remote acceptance separately.

**Latest gate status:** release validation now requires `VelaClipboard_fault_isolation_version() == 1` so an old ABI-1 DLL cannot masquerade as containing fault isolation. The old DLL was correctly rejected. The final strict managed rebuild passed with zero warnings/errors; offline regressions report **163 passed, 0 skipped**, including positive final-VPX validation. Rust static tests passed **2/2** and the Release DLL SHA-256 is `FD3EA8A24328DC0075CCAC2F37DE22AF64A4071D47D98CBD3A0741EA5BCC5BAF`. GitHub workflows and real remote sessions remain unrun.

## 1. Reference conclusions

This plan references 1Remote at commit `f1cfdd1fc518c236e5bd339d3f0b74452f8f9303`. 1Remote uses the Windows `MsTscAx` ActiveX control and configures redirection through `RedirectClipboard`, `RedirectDrives`, `DriveCollection`, and dynamic-device notifications. Clipboard negotiation, file clipboard transfer, and drive access are implemented by the Microsoft RDP control. This repository continues to use IronRDP, does not embed ActiveX, and does not copy 1Remote's GPL-3.0 source.

Microsoft treats clipboard and drive redirection as separate settings, although a server policy that blocks drive redirection also prevents file transfer through the clipboard. That policy coupling does not mean a protocol implementation must share an entire disk. File clipboard transfer is the MS-RDPECLIP descriptor, contents request/response, and file-stream mechanism; Unicode text is not a substitute.

References:

- [1Remote redirection setup](https://github.com/1Remote/1Remote/blob/f1cfdd1fc518c236e5bd339d3f0b74452f8f9303/Ui/View/Host/ProtocolHosts/AxMsRdpClient09Host.xaml.cs#L318-L374)
- [1Remote RDP redirection properties](https://github.com/1Remote/1Remote/blob/f1cfdd1fc518c236e5bd339d3f0b74452f8f9303/Ui/Model/Protocol/RDP.cs#L821-L840)
- [Microsoft RedirectClipboard property](https://learn.microsoft.com/en-us/windows/win32/termserv/imsrdpclientadvancedsettings5-redirectclipboard)
- [Microsoft clipboard redirection policy](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-clipboard)
- [MS-RDPECLIP file stream](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpeclip/6119fce5-72b8-4ec3-8871-f0b785ededd3)

## 2. Current implementation boundary

`VelaShell.Plugin.Rdp` currently provides:

- Bounded Unicode text CLIPRDR: a 2 MiB text limit, bounded format-list and outbound queues, focus and session-epoch isolation, and a Ctrl+V barrier that waits for the format-list acknowledgement.
- Static Windows x64 RDPDR logical-drive mapping: disabled by default and read-only by default, with configuration such as `D:\;E:\`; this exposes a complete logical-drive root, not a file or directory.

It currently does not provide:

- image, HTML, or rich-text clipboard data;
- copying a file in local Explorer and pasting it remotely, or the reverse file clipboard flow;
- directory-level sharing, UNC mappings, or dynamic hot-plugging;
- a guarantee of real Windows server interoperability. Offline state-machine, native ABI, headless, and settings tests do not replace real remote acceptance.

File clipboard and drive mapping must remain separate settings. Enabling text clipboard must not implicitly grant local file reads; enabling a whole-drive mapping must not be described as file clipboard support.

## 3. Phased implementation

### Phase 1: failure isolation and reproducible delivery

First modify `Ui/RdpClipboardBinding.cs`, `Ui/RdpInputQueue.cs`, `Engine/RdpSession.cs`, `Engine/RdpClipboardState.cs`, and the existing error-banner path: clipboard rejection, timeout, size rejection, or local clipboard contention cancels the current paste or pauses clipboard support without disconnecting the desktop. Actual input-send failures, socket-write failures, and unrecoverable native-state corruption remain session failures.

Cancellation must carry an operation generation so that an already queued operation or a late result cannot complete a later paste. Preserve the no-wire-ID request slot and acknowledgement-draining rules; do not fake recovery by disabling and re-enabling the channel in place. Focus loss, unload, and disconnect actively cancel preparation while matching key-up and `ReleaseAll` remain deliverable.

At the same time, build the fixed IronRDP native extension before CI/release tests and verify the final VPX for ABI, digest, RID, and license contents. `native/artifacts` is a local build output and cannot be a clean-checkout release prerequisite.

Acceptance: clipboard failure never calls `DisconnectAsync`; the current paste emits no partial key sequence; later ordinary input still works; real transport failures still disconnect; the final package contains the win-x64 extension.

### Phase 2: active-session text change notifications

Reuse `RdpClipboardBinding`'s epoch, coalescing, and `TopLevel.Clipboard`. On Windows, first evaluate Avalonia's public `Win32Properties.AddWndProcHookCallback` / `RemoveWndProcHookCallback`; listen only while the desktop is active rather than globally polling every tab. Remote writes must carry source/generation information to suppress feedback loops; change notifications and the Ctrl+V barrier remain separate. Keep an explicit focus/key-driven boundary on non-Windows platforms.

Acceptance covers updates by an external process, context-menu paste, `Shift+Insert`, switching between two sessions, notification bursts, write-back feedback, view destruction, and reattachment.

### Phase 3: Windows file clipboard

First extend the MS-RDPECLIP file sub-state machine, then add a Windows OLE/system-clipboard adapter. The protocol requires file descriptors, range-based file-content requests/responses, chunking, bounded concurrency, cancellation, locks, and clipboard generations; announcing Unicode text alone is insufficient.

Start with bidirectional single-file copy, then extend to multiple files and directories; the first version copies without deleting sources. Bound names, reparse points, traversal, sizes, offsets, and file handles. A remote file index must never be allowed to resolve arbitrary local paths.

File clipboard cannot simply reuse text's “focus loss invalidates everything” rule: the user must be able to leave the desktop and paste into local Explorer. The active session stops publishing new announcements, while an already published object remains bound to its source session, clipboard generation, and file allow-list; retire it when the clipboard is replaced, the session disconnects, the view is disposed, or its lease expires. COM callbacks must not synchronously block the UI or network pump.

Add `redirectFileClipboard`, defaulting to false and independent from `redirectClipboard` and `redirectDrives`. Validate ordinary files before deciding directory and large-file behaviour; do not claim image/HTML support as part of this phase.

### Phase 4: directory mappings and dynamic devices

First audit the pinned WindowsRdpdrBackend's root handles, relative-path handling, and read-only enforcement, then extend directory roots. Keep the existing complete-drive-root configuration compatible; use an explicit directory root, share name, and read-only property for new mappings, and do not replace handle isolation with string-prefix checks. Design device authorization, pending-I/O cancellation on removal, reinsertion, and disconnect lifecycle separately for dynamic devices. Continue rejecting UNC unless its boundaries are independently verified.

## 4. Verification and release gates

Every phase must rebuild before running `VelaShell.Plugin.Rdp.Tests`, excluding `ManualPerformance` for the normal regression suite, and run the strict non-incremental build with runtime-async disabled. Await into a local variable before collection/span assertions. Native tests must cover ABI, extension presence, and final VPX extraction; a test that passes when the extension is missing is not a delivery gate.

Run real-server tests only after a target and authorization are explicit: bidirectional text, server policy rejection, actual RDPDR I/O/read-only rejection, single/multiple-file clipboard copy, cancellation, disconnect, and large files. Mark unrun cases as unverified.

The plugin README remains the developer-facing record of current facts in the implementation repository; this plan's Chinese and English mirrors must stay synchronized. Handle commits and pull requests under repository policy; do not commit automatically as part of this plan.
