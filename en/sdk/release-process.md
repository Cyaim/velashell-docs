# Release process (contract SDK)

> This page covers how **this repository** is released. For `vela-plugin` / `VelaShell.PluginSdk.Build` see
> [velashell-plugin-cli](../cli/release-process.md);
> for the `dotnet new` templates see
> [velashell-plugin-templates](../templates/release-process.md).

One release of this repository produces two packages, which share the version number in the Release tag:

| Package | Contents |
| --- | --- |
| `VelaShell.PluginSdk` | The contract assembly |
| `VelaShell.PluginSdk.Testing` | Test doubles |

**Downstream does not have to release along with it.** Since the repository split (2026-08-27) the cli and
templates repositories each have their own version number, and they raise the `VelaShell.PluginSdk` version they
reference only when they want the new contract. The SDK shipping 1.6.0 does not mean `vela-plugin` has to become
1.6.0 — that is exactly what the split was for.

---

## 1. How to release

Three steps:

1. Only for a **breaking change**: bump `VelaPluginApi.Level` in `src/VelaShell.PluginSdk/VelaPluginApi.cs`
   by one, by hand. The script checks it but **does not change it for you** — whether the contract broke is a
   human judgement, not something inferred from a version number.

2. Stamp the version locally and merge it into `main` together with the feature changes:

   ```powershell
   pwsh scripts/Set-Version.ps1 1.6.0
   ```

3. **Publish a Release** on GitHub with the tag `v1.6.0` (with or without the `v` — the pipeline does a
   `TrimStart` — but using it consistently is recommended). For a pre-release, tick prerelease and use a tag
   such as `v1.6.0-preview.1`.

Once the pipeline has parsed the tag, the **first thing** it does is run `Set-Version.ps1` as well. That changes
only the working tree on the runner and **is never written back to the repository** — so the artifact version
always equals the Release tag, whatever the repository happened to have committed at the time.

### If you forgot to stamp the version before releasing

This release is not affected (the Stamp step already covered it), but `main` is now behind: the CI "Version
consistency check" on `main` goes red once. Run `Set-Version.ps1` locally with the command it prints and merge
a follow-up PR.

### Re-running by hand

The `release` event occasionally does not fire. Actions page → pick the Release workflow → Run workflow →
enter the tag. Pushes use `--skip-duplicate`, so re-running for the same tag is idempotent. To validate without
pushing, tick `dryRun`.

---

## 2. Setting up NuGet Trusted Publishing

No API key is stored for pushing: the workflow takes this run's GitHub OIDC token to nuget.org and exchanges it
for a temporary key valid for one hour. On the nuget.org side a **policy** decides "which workflow in which
repository may push packages on my behalf".

⚠️ **After the split you need three policies, one per repository** — a policy matches on (owner, repository,
workflow file), and all three of those differ between the three repositories. A policy's owner covers **every**
package under that account, so you do not need one per package.

This repository's policy:

| Policy field | Value |
| --- | --- |
| Policy name | `velashell-plugin-sdk` (anything you will recognise) |
| Policy owner | `joes_du` |
| Repository Owner | `VelaShellLabs` |
| **Repository** | `velashell-plugin-sdk` |
| **Workflow File** | `release.yml` — **the file name only**, without the `.github/workflows/` prefix |
| Environment | Leave empty (the workflow does not use GitHub Environments) |

To create it: sign in to nuget.org → your user name at the top right → **Trusted Publishing** → **Add**.

While you are there, delete the pre-split policy that points at `velashell-plugin-toolchain` — that repository
no longer pushes packages, and keeping the policy only leaves an extra piece of trust surface.

### ⚠️ A new policy has a 7-day window

A policy newly created on a private repository is "**temporarily active**": it must be used for a successful
publish within 7 days or it lapses automatically (the window can be reopened at any time). The reason is that
nuget.org records the GitHub repository ID and owner ID into the policy on the first successful publish, pinning
it to that repository (a defence against resurrection attacks of the "delete the repository and recreate one with
the same name" kind) — and without a real publish it cannot get those two IDs. So **publish once as soon as the
policy is created**, even if it is only a preview.

Policies on public repositories are usually permanently active straight away, but publishing once to check is
still the low-effort option.

### When no key comes back, check these three first

When the `NuGet login` step fails, nine times out of ten the policy does not match:

* Repository still says `velashell-plugin-toolchain`;
* Workflow File was entered as `.github/workflows/release.yml`;
* `NUGET_USER` was set to an email address — it must be the nuget.org **user name** (profile name).
  The workflow reads `vars.NUGET_USER` by default and falls back to `joes_du` when it is not set.

Also, the job must keep `permissions: id-token: write`; without it GitHub does not issue an OIDC token at all.

---

## 3. Version discipline

`VelaSdkVersion` in `Directory.Build.props` is this repository's default version; at release time the tag
overrides it.

### Three version numbers, three jobs — do not mix them up

| Property | Value | Role |
| --- | --- | --- |
| `AssemblyVersion` | `<major>.0.0.0` | The **binding identity**; it moves only with the major version. Plugins bind to this identity at compile time |
| `FileVersion` | The full numeric version (`1.6.0`) | The one shown on the Explorer properties page |
| `InformationalVersion` | The full version including any pre-release suffix | The version `vela-plugin` reports |

`AssemblyVersion` is pinned to the major version because letting it follow the patch number would make every
compiled plugin rebind on every patch release, for no benefit. A new major version, on the other hand, means the
contract broke — and at that moment **apiLevel must be bumped by one as well**, so an older host cleanly refuses
the plugin by apiLevel at **discovery time** instead of throwing an incomprehensible binding exception at load
time.

### "SDK major version == apiLevel"

`Set-Version.ps1` enforces this rule: to release `2.x.x` you must first change `VelaPluginApi.Level` to `2`, and
vice versa. apiLevel is a **property of the contract**, so this check only has a home in this repository —
neither the CLI nor the templates are allowed to touch it.

### Where the version number lives (maintained by the script)

After the split only four places are left in this repository:

| Location | What goes wrong if it is missed |
| --- | --- |
| `VelaSdkVersion` in `Directory.Build.props` | The package version is wrong |
| `SdkVersion` in `src/VelaShell.PluginSdk/VelaPluginApi.cs` | **Nothing reports an error**; `vela-plugin doctor` simply reports the wrong host SDK version from then on, and plugins' `minSdkVersion` gate is judged wrongly with it |
| The version banner in `docs/sdk-reference.md` | A stale number for people to copy |
| The version banner in `docs-en/sdk-reference.md` | Same as above |

`pwsh scripts/Set-Version.ps1 <version>` changes them all in one go. CI runs it with `-Check` as a health check.

---

## 4. The Avalonia version pin: this repository is the authority

The Avalonia a plugin project compiles against and the host's Avalonia it is forced to share at run time inside
the host process must be the same version — the loader makes every `Avalonia*` assembly fall back to the loading
side, so a version drift means control types that do not match across ALCs, and it only blows up once a user
installs the plugin.

In the single-repository days, `VelaShell.PluginSdk.Build` made this check by `XmlPeek`-ing the host's
`src/Directory.Packages.props` directly. After the split neither side can read the other's files any more, so it
now runs the other way round:

```
Directory.Build.props: <VelaAvaloniaVersion>12.1.3</VelaAvaloniaVersion>
        ↓ written into buildTransitive/VelaShell.PluginSdk.props at pack time
$(VelaSdkPinnedAvaloniaVersion)
        ↓ visible to every project that references VelaShell.PluginSdk
  ├── VerifyAvaloniaMatchesSdk in the host's src/Directory.Build.targets
  └── VerifyAvaloniaVersionPin in VelaShell.PluginSdk.Build (cli repository)
```

That props file in the package is the **only channel** that carries this fact downstream, and leaving it out of
the package does not make `pack` fail — so both CI and Release have a "Verify the pin props shipped in the
package" step that unpacks the nupkg and checks it.

**The full procedure for changing `VelaAvaloniaVersion`**:

1. Ship it in the same wave as the host (the host must reference the same version);
2. Release the SDK;
3. In the cli repository, raise `VelaSdkPackageVersion` to that release and update its `VelaAvaloniaVersion` to
   match — otherwise its `VerifyAvaloniaVersionPin` goes red;
4. Release `VelaShell.PluginSdk.Build`; only then do plugin projects get the new pin.

---

## 5. Why the tests must run in Debug

`src/Directory.Build.props` turns on strong-name signing in Release (the host is signed, and a signed assembly
cannot reference an unsigned one, so the SDK has to be signed with the same key as the host). On a signed
assembly, `InternalsVisibleTo` must carry the friend's public key, and the friend itself must be signed with the
same key — the test assemblies meet neither condition.

These tests check the contract and do not depend on the optimisation level, so Debug is equivalent; and Debug's
`obj/bin` are kept apart from the Release directories `pack` uses, so unsigned output never gets mixed into the
package being published.

The key is not checked in (`*.snk` is in `.gitignore`); CI restores it to the repository root from the
`STRONG_NAME_KEY` secret. It is the **only** secret this repository needs.

> Neither the cli nor the templates repository signs anything, so they **do not need** this key —
> an unsigned assembly may reference a signed one, which is the right direction.
