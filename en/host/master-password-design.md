# Optional Master Password: Design Draft

> Date written: 2026-10-25　　Baseline code: host `dev` (the first step of master password protection is done: the local encryption key is handed to the system keychain)
>
> Status: **draft**; implementation waits until the maintainers settle the decision points in §9. Not implemented in this round.
>
> Related: [System-keychain and sudo credential fill feasibility research](system-keychain-and-sudo-credential-fill-feasibility-research.md) (its 2026-10-25 erratum at the top describes how the first step was implemented),
> [Credential manager integration design](credential-manager-integration-design.md) §5.7 (interactive / non-interactive resolution), [settings-audit](settings-audit.md) R-04.

## 0. Conclusion first

- The master password is an **optional** second layer, off by default. Once on, what the system keychain (macOS keychain / Secret Service) or the Windows DPAPI file holds
  is no longer the local encryption key itself but **the local key wrapped with a key derived from the master password**: both the keychain entry and the master password are needed to open it.
- **Existing `enc1:` ciphertexts do not change by a single byte.** The master password wraps the local key (the data encryption key, DEK), not each ciphertext — turning the master
  password on, off or changing it rewrites only a few dozen bytes of wrapping, takes seconds and never touches the database. KeePass / Bitwarden / 1Password do the same.
- Unlocking: an unlock dialog appears once after start-up (cancellable); after cancelling, or while locked, the app keeps working — saved passwords just cannot be read. Automatic reconnects, tunnels
  started automatically, automatic sync and plugins reading secrets in the background all treat it as "locked" and **never show the dialog**.
- **Forgetting the master password = the saved secrets are void; there is no back door.** A "Reset" is offered: create a new key and enter passwords again; the old wrapping and the old
  ciphertexts in the database are all kept (they can still be opened if the password comes back to mind).

## 1. Where the first step left things

- `MasterKeyResolver` resolves a **key ring** (one or more 32-byte DEKs; the first encrypts, decryption tries each in turn); on macOS / Linux each key is one keychain entry
  (id = the first 128 bits of a domain-separated SHA-256(DEK)), and `secret.key.ref` in the data directory lists only the ids; on Windows it is the DPAPI-wrapped `secret.key`; when the keychain is unavailable it falls back to a plaintext `secret.key`.
- `ISecretProtector` is a synchronous `string → string`, with callers all over the session repository, shared credentials, Gist, plugin secrets and the plugin trust store; `TryUnprotect` can say truthfully that something could not be decrypted.

## 2. Threat model: what a master password adds

| Scenario | First step (keychain) | With a master password |
| --- | --- | --- |
| The data directory is copied / backed up / synced to a cloud drive | Stopped | Stopped |
| Another program running as the same user reads the keychain entry (macOS entries are open to any application; Secret Service is open to processes in the session once unlocked; DPAPI to the user's own processes) | **Not stopped** (same level as the former 0600 file) | Stopped (the master password is needed too) |
| Keychain / keyring files leak together with the login password; DPAPI after a domain administrator resets the account | Not stopped | Stopped |
| Someone sits down at your logged-in, unlocked computer and opens VelaShell | Not stopped | Stopped while locked (§5 automatic locking) |
| Reading process memory or keylogging after unlocking | Not stopped | **Not stopped** (stated plainly, not promised) |
| In-process plugins (same process as the host) | Not stopped | Not stopped after unlocking; isolated plugins still reach only their own secrets through `ISecretsApi` |

## 3. Key hierarchy and storage format

```
master password ──Argon2id(salt, m, t, p)──► KEK (32 bytes, zeroed as soon as wrapping / unwrapping is done)
                                              │ AES-256-GCM (random 12-byte nonce)
DEK (every key in the first step's ring) ──────►  wrapped = nonce ‖ tag ‖ GCM(KEK, DEK)
```

- The **KDF header** (salt and parameters, not secret) lives in `secret.key.mp` in the data directory: `version=1`, `kdf=argon2id`, `m`, `t`, `p`, `salt` (16 random bytes, base64).
  The whole key ring shares one KEK (one derivation opens them all).
- The value of **each keychain entry** changes from "32-byte DEK" to the magic `VSMP1` ‖ `wrapped` (60 bytes). The entry id is still derived from the DEK and does not change —
  so the first step's rule "an entry whose content does not match its id is neither used nor touched" protects wrapped entries for free: an older version that does not know `VSMP1` only reports "not found" and creates a new key, never overwriting (a downgrade loses no data).
- **Windows**: the content of `secret.key` changes from `DPAPI(DEK)` to `DPAPI(VSMP1 ‖ wrapped)`; the same goes for platforms that fell back to a plaintext file (`VSMP1 ‖ wrapped`, where the master password is then the only barrier).
- Verification: the GCM authentication tag is the master password check; no separate verifier is stored (one less thing to compare against offline).
- **The source of truth is the wrapping format itself**, not `AppSettings.MasterPasswordProtection`: settings import and Gist sync can both change that boolean.
  The field is used only for display, and is registered in `SettingsCategories` as a "not exported, not synced" field.
- Alternative (not recommended): with the master password on, keep the key only as a wrapped file in the data directory and delete the keychain entries — the data directory plus the master password would move to another machine,
  but with a weak master password this falls back to "get the directory and brute-force it offline", the opposite of what the first step is for; moving machines already has connection export and end-to-end sync.

## 4. KDF: Argon2id

- Parameters: m = 64 MiB, t = 3, p = 1 (the second recommendation of RFC 9106 §4 is m = 64 MiB, t = 3, p = 4; the OWASP minimum is m = 19 MiB, t = 2, p = 1), about 0.3–0.6 s on a mainstream laptop.
  Parameters are stored in the header; later tuning re-wraps with new parameters and a new salt on the next successful unlock (rehash), with no one-off migration.
- Implementation: the BCL has no Argon2. The host already carries BouncyCastle.Cryptography through `VelaShell.Ssh` (it provides Argon2id for `.ppk` v3), so Infrastructure references the same package explicitly and adds no dependency;
  the comment in `src/Directory.Packages.props` records this new use (the "no second use" in `VelaShell.Ssh/README.md` refers to the inside of the SSH library).
- Derivation runs on a background thread (64 MiB of memory, a few hundred ms of CPU), with the unlock dialog in a busy state.
- Online guessing: after repeated wrong entries the unlock dialog waits a little longer each time (1, 2, 4… s, capped at 30 s); nothing is locked out or deleted. Offline guessing is held back only by the parameters and password strength: turning it on suggests a length ≥ 12 or a passphrase.

## 5. Unlocking, the locked state and saving

- **When to unlock** (pending, §9-1): proposed is one unlock dialog after the main window appears, cancellable; after cancelling, a lock icon stays in the status bar to unlock later.
  Reason: blocking start-up would get in the way of someone who only wants a local terminal, and `Unprotect` is a synchronous interface with call sites all over the repository, so a dialog cannot be raised in place at each call site.
- **Semantics while locked**: `AesSecretProtector`'s key ring is empty → `TryUnprotect` returns false and `Unprotect` returns the ciphertext as is (exactly the first step's "key not obtained" path, which callers already handle);
  `Protect` must not create a new key (that would produce a key the master password does not cover) and throws a dedicated `SecretsLockedException` instead.
- **A new asynchronous gate** `IMasterPasswordGate { bool IsLocked; Task<bool> EnsureUnlockedAsync(bool interactive, CancellationToken); event Unlocked; }`.
  Entry points that use secrets (the five build points of the credential manager design §5.5, Gist sync, plugin `ISecretsApi`, plugin trust store writes, saving in the editor) await it before resolving / saving.
  Locked while saving: interactive entry points show the unlock dialog; cancelling means the password is not saved this time (the connection still uses what was typed), with a plain "Not unlocked; this password was not saved".
- **Automatic locking** (pending, §9-2): optional "after N idle minutes" and "when the system locks"; locking = zeroing the DEKs in memory (`CryptographicOperations.ZeroMemory`) and going back to locked. Proposed default: no automatic locking.
- **Unlock dialog**: modal to the main window, `SecurePasswordBox`, never copied to the clipboard or history; serialized (one dialog at a time, other requests wait for its result).

## 6. Automatic reconnects and background tasks

The same table as the [credential manager integration design](credential-manager-integration-design.md) §5.7 (its "vault locked" is "master password locked" here):

| Trigger | Mode | While locked |
| --- | --- | --- |
| A connection started by the user, a manual reconnect (Ctrl+R / Enter / "Reconnect"), connection diagnostics, plugin `Sessions.OpenAsync`, restoring sessions at start-up (user intent, counted as one interaction) | Interactive | Show the unlock dialog |
| The automatic reconnect timer, `ReconnectAllAfterResume` after waking from sleep, tunnel `AutoReconnectAsync` | Non-interactive | No dialog; the tab's failure overlay says "Locked; unlock and press Ctrl+R to reconnect"; after a successful unlock each pending automatic reconnect is retried once |
| Automatic Gist sync | Non-interactive | Skip this round and log a line; a manual "Sync now" is interactive |
| A plugin reading secrets in the background (`ISecretsApi.GetAsync`) | Non-interactive | Needs a new SDK contract, a "locked" exception (written in velashell-plugin-sdk without changing the version); until the contract is ready, wait for an interactive unlock. **Never return null**: the plugin would think the secret does not exist, ask the user to enter it again and overwrite the old value |

Same reasoning as the credential manager design: when the network comes back and 20 tabs reconnect at once, an unlock dialog the user cannot attribute to anything must not pop up (phishing concerns).

## 7. Turning on / off / changing / forgetting

- **Turn on** (Settings → General → Privacy & security, restoring the R-04 row and wiring it): enter twice, strength hint, confirm "If I forget the master password, saved passwords cannot be recovered" → resolve the current key ring → derive the KEK →
  "write the new one, read it back and compare" entry by entry → write `secret.key.mp`. Any failing step returns to "not on" as a whole (an entry is replaced only after its check passes; the first step's "never overwrite a mismatching entry" and atomic writes apply).
- **Turn off**: enter the current master password → unwrap → write each plaintext DEK back (the first step's form) → delete `secret.key.mp`.
- **Change**: old + new → unwrap → re-wrap with a new salt and a new KEK → write the new ones, check, then swap the header. The database is untouched.
- **Forget**: "Forgot the master password?" in the unlock dialog → explain the consequences → after confirming: the old entries and header file are renamed and kept (`.forgotten-<timestamp>`, not deleted), a new DEK is created (optionally set a new master password right away),
  and from then on it follows the first step's "entry not found" notice path: passwords are entered again when connecting. (Pending, §9-3: whether turning it on also creates a recovery code.)
- **Audit**: turning on / off / changing / resetting each records a `settings` audit entry (the action only, never any password material).

## 8. Migration and compatibility

- First step → second step: wrapping happens once, only when the user turns it on; nothing changes for those who don't.
- Downgrading to a version that does not know `VSMP1`: the wrapped entry does not match its id → the old version reports "not found", creates a new key and overwrites nothing; saved passwords cannot be decrypted in the old version,
  and after upgrading again and unlocking everything works (the key the old version created is merged by the first step's rules).
- Multiple devices: the master password only covers this machine; the end-to-end passphrase of cloud sync is a separate thing, neither merged nor reused (so one leaked passphrase does not compromise both).

## 9. Pending decisions

1. When to unlock: once after start-up (cancellable) vs only when a secret is first needed. The draft leans towards the former.
2. Default for automatic locking: no automatic locking vs 15 idle minutes.
3. Recovery code: create a random 128-bit code when turning it on (written down by the user) that can also unwrap the DEKs. Cheap (one more wrapping); leaning towards offering it, skippable.
4. The "locked" semantics of plugin `ISecretsApi` need a new SDK contract.
5. Windows Hello / Touch ID as an unlock method (item 2 of §8 in the [research](system-keychain-and-sudo-credential-fill-feasibility-research.md)): left for a third step.
6. Whether the Argon2id parameters drop to m = 32 MiB on low-end machines (a Linux desktop on a Raspberry Pi).

## 10. Test plan

- Round trips of the KDF header and the wrapping format; a wrong master password → GCM failure → failed-unlock count and back-off.
- Turning on / off / changing interrupted at every step loses none of the DEKs (reusing the first step's fake-keychain test shapes).
- While locked: `Protect` creates no new key, `TryUnprotect` is false, automatic reconnects and automatic sync show no dialog, and interactive entry points show only one dialog.
- Downgrade: a resolver that does not know `VSMP1` does not overwrite a wrapped entry.
- Settings import / Gist sync changing `MasterPasswordProtection` does not affect the actual protection state.
