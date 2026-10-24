# Plugin Contract SDK Documentation

中文:[`../../zh/sdk/`](../../zh/sdk/)

| Document | Contents |
| --- | --- |
| [sdk-reference.md](sdk-reference.md) | **SDK reference**: package layout, entry contract, capability domains, SDK version history, test doubles, load model |
| [release-process.md](release-process.md) | **How `velashell-plugin-sdk` is released**: the Release flow, NuGet Trusted Publishing setup, apiLevel and Avalonia version discipline |

Repository: [velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk)
(`VelaShell.PluginSdk`, `VelaShell.PluginSdk.Testing`).

## Related

Do not start here to write a plugin — read the [dev guide](../templates/dev-guide.md) first;
`sdk-reference.md` is the dictionary you look things up in. For the command line see the
[CLI manual](../cli/cli.md), for shipping see [publishing](../templates/publishing.md).

The plugin system's **architecture blueprint** (process model, IPC protocol, permission system, UI
extensions, threat model, roadmap — documents 01–15) is in [`../plugins/`](../plugins/). That is
host-side design; you do not need it to write a plugin.
