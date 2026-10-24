# Release process (CLI + build-support package)

> This page covers how **this repository** is released. For the contract SDK see
> [velashell-plugin-sdk](../sdk/release-process.md);
> for the `dotnet new` templates see
> [velashell-plugin-templates](../templates/release-process.md).

One release of this repository produces two packages, which share the version number in the Release tag:

| Package | Contents |
| --- | --- |
| `VelaShell.Plugin.Cli` | The dotnet tool `vela-plugin` |
| `VelaShell.PluginSdk.Build` | The one package plugin projects reference |

The two **must be released at the same version**: `.Build` bundles the build output of `vela-plugin` in the
package's `tools/`, and its targets call that packer's command surface (`validate` / `pack`) directly. If the
command-line arguments change and the targets do not keep up, plugin authors see an `Exec` failure halfway through
their build — so the two are one release unit.

**Neither upstream nor downstream has to release along with it.** The contract SDK has its own version number,
and so does the templates package. This repository shipping 1.5.1 does not mean the contract changed, and it does
not require the templates to ship too.

---

## 1. How to release

Two steps:

1. Stamp the version locally and merge it into `main` together with the feature changes:

   ```powershell
   pwsh scripts/Set-Version.ps1 1.5.1
   ```

2. **Publish a Release** on GitHub with the tag `v1.5.1` (with or without the `v` — the pipeline does a
   `TrimStart` — but using it consistently is recommended). For a pre-release, tick prerelease and use a tag
   such as `v1.6.0-preview.1`.

Once the pipeline has parsed the tag, the **first thing** it does is run `Set-Version.ps1` as well. That changes
only the working tree on the runner and **is never written back to the repository** — so the artifact version
always equals the Release tag.

### If you forgot to stamp the version before releasing

This release is not affected (the Stamp step already covered it), but `main` is now behind: the CI "Version
consistency check" on `main` goes red once. Run it locally with the command it prints and merge a follow-up PR.

### Re-running by hand

Actions page → pick the Release workflow → Run workflow → enter the tag. Pushes use `--skip-duplicate`, so
re-running for the same tag is idempotent. To validate without pushing, tick `dryRun`.

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
| Policy name | `velashell-plugin-cli` (anything you will recognise) |
| Policy owner | `joes_du` |
| Repository Owner | `VelaShellLabs` |
| **Repository** | `velashell-plugin-cli` |
| **Workflow File** | `release.yml` — **the file name only**, without the `.github/workflows/` prefix |
| Environment | Leave empty (the workflow does not use GitHub Environments) |

To create it: sign in to nuget.org → your user name at the top right → **Trusted Publishing** → **Add**.

### ⚠️ A new policy has a 7-day window

A policy newly created on a private repository is "**temporarily active**": it must be used for a successful
publish within 7 days or it lapses automatically (the window can be reopened at any time). The reason is that
nuget.org records the GitHub repository ID and owner ID into the policy on the first successful publish, pinning
it to that repository (a defence against resurrection attacks of the "delete the repository and recreate one with
the same name" kind) — and without a real publish it cannot get those two IDs. So **publish once as soon as the
policy is created**, even if it is only a preview.

### When no key comes back, check these three first

When the `NuGet login` step fails, nine times out of ten the policy does not match:

* Repository still says `velashell-plugin-toolchain`;
* Workflow File was entered as `.github/workflows/release.yml`;
* `NUGET_USER` was set to an email address — it must be the nuget.org **user name** (profile name).
  The workflow reads `vars.NUGET_USER` by default and falls back to `joes_du` when it is not set.

Also, the job must keep `permissions: id-token: write`; without it GitHub does not issue an OIDC token at all.

---

## 3. This repository needs no secrets

The assemblies are **not strong-name signed**. That is the right direction: an unsigned assembly may reference a
signed one (`vela-plugin` → `VelaShell.PluginSdk`), just not the other way round. The contract SDK has to be
signed because the **host** (which is signed) references it — that is the sdk repository's business.

Pushing goes through OIDC. So fork PRs and the main branch run exactly the same CI path, and there is no
"degrade when the key is unavailable" branch to maintain.

---

## 4. Two cross-repository knobs that `Set-Version.ps1` deliberately leaves alone

They are not "version numbers"; they are **dependency decisions**:

### The `PackageReference` to `VelaShell.PluginSdk` — which contract SDK release to reference

It lives in two csproj files, `src/VelaShell.Plugin.Cli` and `src/VelaShell.PluginSdk.Build`, and **both must be
at the same version**. It is deliberately written as a literal rather than pulled out into an MSBuild property in
`Directory.Build.props`: `Version="$(...)"` stops Dependabot and `dotnet add package` from recognising the
dependency, and they are exactly what this package is meant to be updated with — merging Dependabot's PR is the
usual way to raise it, and it raises both places together.

Raising it = making the plugin authors' **compile-target contract** newer. A patch release that only changes
`vela-plugin`'s output format should not swap this out on the side — so it is a separate decision that needs to be
thought through.

After raising it, remember to:

1. Build once, so `VerifyAvaloniaVersionPin` checks whether the Avalonia version pin has to move as well;
2. Release this repository's packages; only then do plugin authors get it.

Not raising it is perfectly normal too — the old contract still packs and validates fine.

### `VelaAvaloniaVersion` — the Avalonia version pinned for plugin projects

**The authority is in the sdk repository; this is only a copy that must equal it.** Why keep a copy instead of
using the authoritative value directly: the authoritative value comes from the `obj/*.nuget.g.props` generated
after NuGet restore, while the version range of Avalonia's `PackageReference` is an input that **restore itself**
reads — on a cold restore the value is still empty, the range becomes `[]`, and restore fails outright.

So it is "write a literal + check at build time". The check covers three places:

| Location | What it is |
| --- | --- |
| `$(VelaSdkPinnedAvaloniaVersion)` | The **authority**, from the `buildTransitive` props of the `VelaShell.PluginSdk` package |
| `$(VelaAvaloniaVersion)` | The copy in this repository's root `Directory.Build.props`; the exact range `[x.y.z]` is built from it |
| The default of the same name in `build/VelaShell.PluginSdk.Build.props` | The baseline for the `VELA1001` check on the plugin-project side |

Drift is reported as:

| Error code | Meaning |
| --- | --- |
| `VELA1000` | The two copies inside the repository disagree |
| `VELA1005` | The referenced SDK package is too old to export the authoritative value — the whole cross-repository check chain would be switched off silently, so it fails outright |
| `VELA1006` | The copy disagrees with the authoritative value (cross-repository drift) |

---

## 5. After a release: does the templates repository need to change?

That depends on whether you want projects generated by `dotnet new velaplugin` to point at the new `.Build`.

Yes → go to [velashell-plugin-templates](https://github.com/VelaShellLabs/velashell-plugin-templates),
raise `VelaBuildPackageVersion` and release the templates again.

No → nothing to do. Newly created projects simply keep referencing the previous `.Build` package, which is
entirely usable — this is the normal state of things after the split, not an omission.

---

## 6. End-to-end smoke test

Before a release, `scripts/Invoke-Smoke.ps1` runs: with the freshly packed packages, it copies the hand-written
minimal plugin project under `tests/smoke/` into a temporary directory, restores from the local source, builds,
runs `-t:PackVpx`, reads the container back and checks whether any shared assembly leaked into the plugin output
directory.

**This step is not a formality.** A plugin project is an environment outside the repository and inherits none of
the repository's `Directory.Build.props` (the two empty `Directory.Build.props`/`.targets` in the fixture are
there to cut off the upward search). Historically it is what caught two pitfalls first:

* The SDK carries `RequiresPreviewFeatures`, so a plugin project that does not turn on the preview switch gets
  `CA2252` across the board;
* NuGet does not flow `build` assets transitively by default, so Avalonia's AXAML compiler never reaches the
  plugin project — the symptom is controls that cannot be found at run time, not a build error.

Both pitfalls only show up when nothing at all is inherited.

> Before the split this step used `dotnet new velaplugin-ui`. With the templates moved out it can no longer do
> that: a problem in the templates repository would turn this repository's CI red for no reason, and releasing
> this repository's packages would first require a templates package. End-to-end verification of the templates
> themselves is the job of the templates repository's own CI.
