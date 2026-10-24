> **This report describes the built-in implementation; S3 became a plugin on 2026-08-14.**
> For the current architecture and trade-offs see [`s3-protocol-plugin-design.md`](s3-protocol-plugin-design.md); this document is kept as a record of the decisions.

# Full S3 Protocol Support: Implementation Report

> Written: 2026-08-14　　Baseline code: `main` @ `ab8a9fa`
>
> Purpose: record the **scope correction, technical decisions, coverage list and verification results** of this "full S3 protocol support" work.
> The design and its trade-offs are worked out in [`s3-protocol-plugin-design.md`](s3-protocol-plugin-design.md); this document is a delivery note for reviewers and does not repeat design details.
>
> Related: [`ftp-client-feasibility-research.md`](ftp-client-feasibility-research.md) (this work reuses the `ISftpService` seam it established)

---

## 1. Scope correction (where this work started)

The first version built "S3 support" around 15 operations (list / upload / download / delete / copy), and on that basis
concluded that "hand-signing SigV4 in 200 lines is cheaper than bringing in the SDK".

This time the real protocol surface of `IAmazonS3` was counted by reflection first:

| Category | Count |
|---|---|
| Object reads, writes and transfer | about 20 |
| **Bucket-level configuration** | **about 55** |
| Object-level properties | about 20 |
| Other (presigning, S3 Select, metadata tables, etc.) | about 20 |
| **Total** | **116** |

**Conclusion: the earlier build-vs-buy comparison had the wrong denominator.** Covering 116 operations means rewriting
XML serialisation and error-code mapping once per operation, and following along every time AWS adds a capability. The
first version has been torn down and redone.

> **Reproducible**: the count was made by loading `AWSSDK.S3` and reflecting over all `*Async` methods of `IAmazonS3`,
> de-duplicated — not from memory or a documentation index. Doing the same is recommended when evaluating similar
> dependencies later: measure the protocol surface first, then talk about "two hundred lines vs. one package".

---

## 2. Technical decision

**AWSSDK.S3 4.0.102.1 is adopted** (Apache-2.0, compatible with this repository's AGPL-3.0 + commercial dual licence).
The hand-written SigV4 / REST stack (`AwsSignatureV4`, `S3RestClient`, `S3Xml`) has been deleted.

The cost is two more assemblies (AWSSDK.S3 + AWSSDK.Core, about 1 MB each). The benefit is that all 116 operations are
reachable, and concurrency / retries / aborts of multipart uploads are handled by `TransferUtility`.

**The hard layering rule is unchanged**: AWSSDK types (`AmazonS3Exception`, `S3StorageClass` and so on) **must not
cross the Infrastructure boundary**, and translation is concentrated in one place, `S3Interop`; Core only sees the
neutral contracts under `VelaShell.Core.S3` and the `VelaS3*` exception family. This is the same convention as for
FluentFTP and Tmds.Ssh.

---

## 3. Architecture

Two service tracks, sharing the same session and the same `IAmazonS3` client (through `IS3ClientAccessor`):

| Component | Contract | Responsibility |
|---|---|---|
| `S3FileService` | `ISftpService` + `IS3SessionService` | File-manager semantics: flat key space → directory tree |
| `S3ManagementService` | `IS3ManagementService` | Every other capability: bucket configuration, versions, ACLs, object lock, tags, archive restore, multipart-upload management, S3 Select |

Because the seam is still `ISftpService`, **dual-pane browsing, the transfer queue, rate limiting, drag and drop and
conflict policies all work with zero changes**.

**Bucket configuration is collapsed into one set of triple methods** + dispatch on `S3ConfigKind`, instead of twenty-odd
sets of methods of their own (which would bloat the interface to more than seventy members):

```csharp
Task<S3ConfigResult> GetBucketConfigAsync(session, bucket, S3ConfigKind kind, id?);
Task PutBucketConfigAsync(session, bucket, S3ConfigKind kind, string json, id?);
Task DeleteBucketConfigAsync(session, bucket, S3ConfigKind kind, id?);
Task<IReadOnlyList<string>> ListBucketConfigIdsAsync(session, bucket, S3ConfigKind kind);
```

Adding a kind of configuration = **one enum member + one switch branch + one navigation entry**.
That is the structural reason the coverage can be "complete" rather than "half".

---

## 4. Operation coverage: 112 / 116

The 4 not implemented are all operations a client cannot or should not call:

| Operation | Reason |
|---|---|
| `WriteGetObjectResponse` | Can only be called from inside an S3 Object Lambda function; a desktop client physically cannot use it |
| `CreateSession` | The internal session token of S3 Express directory buckets, managed automatically by the SDK |
| `GetObjectTorrent` | BitTorrent distribution, deprecated by AWS |
| `ListDirectoryBuckets` | Exclusive to S3 Express One Zone; this feature targets general-purpose S3 and compatible implementations, and it will be added when there is demand |

`InitiateMultipartUpload` / `UploadPart` / `CompleteMultipartUpload` / `CopyPart`
are not exposed directly in the UI — the transfer engine uses them internally; **management** of multipart uploads
(listing, aborting) can be done in the bucket manager.

---

## 5. UI (all of it operable)

### 5.1 S3 bucket manager (`S3ManagerWindow`)

Left-hand navigation = Overview + Object versions + Incomplete multipart uploads + **all 22 bucket configuration
items**; the right-hand side shows a structured form / JSON document editor / dedicated list according to type.
Navigation is driven by `S3ConfigDescriptor.All`: **whatever the protocol has, this has.**

| Page | Operations covered |
|---|---|
| Overview | `GetBucketLocation`, `GetBucketPolicyStatus`, `GetBucketVersioning`, `GetObjectLockConfiguration` |
| Object versions | `ListVersions` + deleting a given version + restoring an earlier version (implemented as a copy) |
| Incomplete multipart uploads | `ListMultipartUploads` + `AbortMultipartUpload` |
| 22 bucket configuration items | Versioning, lifecycle, bucket policy, public access block, object ownership, bucket ACL, CORS, default encryption, object lock, bucket tags, replication rules, static website, access logging, event notifications, transfer acceleration, requester pays, inventory, analytics, request metrics, intelligent tiering, metadata tables, ABAC |

### 5.2 S3 object inspector (`S3ObjectInspectorWindow`)

Tabs: Details / Tags / Permissions / Retention and legal hold / Storage and encryption (including archive restore) /
S3 Select. Covers `GetObjectAttributes`, `GetObjectMetadata`, `Get/Put/DeleteObjectTagging`,
`Get/PutObjectAcl`, `Get/PutObjectRetention`, `Get/PutObjectLegalHold`, `RestoreObject`,
`CopyObject` (changing storage class / encryption / metadata), `SelectObjectContent` and `GetPreSignedURL`.

### 5.3 File browser

Three S3-only context-menu items were added (Copy share link, S3 object management, Bucket management); they do not
appear under other protocols.

### 5.4 Editor strategy (a deliberate trade-off)

| Type | Suited to | Items |
|---|---|---|
| Structured form | Few fields, stable shape | 10 (versioning, public access block, object ownership, encryption, object lock, tags, static website, access logging, transfer acceleration, requester pays) |
| JSON document editor | **Things that are documents to begin with** | 12 (lifecycle, bucket policy, ACL, CORS, replication, event notifications, inventory, analytics, metrics, intelligent tiering, metadata tables, ABAC) |

Forcing the second group into forms would not only be incomplete, it would also **break the user's existing
configuration on meeting an unknown field** — a field read in but not rendered is gone once serialised back. The AWS
console also gives JSON for these items.

Even for the form-based items, writing back goes through incremental edits on the JSON DOM (`S3ConfigForm.Apply`),
overwriting only the paths the form actually edited and keeping unknown fields as they were. That property is pinned
down by `S3ConfigFormTests.Apply_PreservesFieldsTheFormDoesNotKnowAbout`.

---

## 6. Compatibility notes

The three most likely to go wrong on real servers, all handled in `S3ClientFactory`:

1. **`RequestChecksumCalculation` set to `WHEN_REQUIRED`** — AWSSDK v4 computes a CRC32 for every request by default
   and expects responses to carry a checksum back; older MinIO / Ceph RGW and gateways of all kinds do not understand
   these headers, which shows up as uploads rejected outright or checksum failures on download.
2. **`AllowAutoRedirect = false`** — region redirects have to be re-signed and re-sent by the SDK according to
   `x-amz-bucket-region`; following them automatically takes **the old region's signature** to the new address, which
   is bound to produce `SignatureDoesNotMatch`.
3. **`Auto` addressing uses virtual-hosted style only for `*.amazonaws.com`**, and path style for everything else —
   self-deployed MinIO / Ceph does not resolve `bucket.host` by default (it needs `MINIO_DOMAIN`), whereas path style
   works on both.

The rest:

- **"Never configured" and "unsupported" are recognised as empty states, not errors** (`S3Interop.IsMissingOrUnsupported`).
  AWS has a dedicated error code for every capability that has not been configured, while compatible implementations
  often just return 501; treating them as errors would turn the bucket manager red the moment it opens.
- **Self-signed certificates** reuse FTPS's "reject first → prompt → record the fingerprint and reconnect" flow and the
  same set of `Cert_*` strings.
- **The part count is capped at 10000**: the part size is raised to `max(configured value, file size / 10000)`.
- **The content type can only be set at the step that initiates the multipart upload** (neither `UploadPart` nor
  `Complete` gets another chance to supply it).

---

## 7. Verification results

**Build: 0 warnings, 0 errors** (whole solution, .NET 11 preview SDK).

**Tests: 1664 passed / 1 failed** (the failure is covered in section 9; it is a pre-existing baseline problem).

| Test suite | Count | What it guards |
|---|---|---|
| `S3FileServiceIntegrationTests` | 31 | End to end against `LoopbackS3Server`: directory folding, paging, placeholder hiding, byte-for-byte identical multipart uploads, content type, Range resume, server-side copy, recursive delete, non-empty buckets not emptied, presigned URLs |
| `S3ManagementServiceTests` | 20 | Mapping and **error classification** on the 116-operation side: empty state vs. unsupported vs. real failure; restoring a version must be implemented as a copy; changing storage class is a self-copy; metadata must carry REPLACE |
| `S3ObjectPathTests` | 13 | Round-trip consistency of path ↔ bucket/key, adding the trailing slash to prefixes, bucket-name validation |
| `S3InteropTests` | 12 | The boundary from AWSSDK exceptions → the Core neutral exception family |
| `SigV4VerifierTests` | 11 | The correctness of **the test infrastructure itself** (checked segment by segment against AWS's 5 official examples) |
| `S3ConfigFormTests` | 11 | **The form must not swallow unknown fields**, nested paths are filled in automatically, an empty value writes null, numbers are written as JSON numbers |
| **S3-specific subtotal** | **98** | |

### 7.1 Why the loopback server still verifies signatures

Signing on the product side is now the SDK's job, but `LoopbackS3Server` recomputes SigV4 from the **raw request line**
it receives and compares, guarding the other half: **whether the client configuration (endpoint / region / addressing
style / credentials) actually reached the wire correctly.** On a real server this class of error only gets the answer
`SignatureDoesNotMatch`, with no clue whatsoever.

The verifier is itself the yardstick, so it must be verified too — which is exactly why `SigV4VerifierTests` is kept.

### 7.2 Real defects found and fixed while writing the tests

| Defect | Impact |
|---|---|
| Multipart uploads lost the object's content type | Every `.html` / `.png` above the threshold degraded to `application/octet-stream`, so a shared link would turn into a file download |
| Blank tag keys were not dropped when saving the form | An all-whitespace key is rejected by the server, and the user sees a pointless failure |
| Deleting a bucket emptied it on the user's behalf | **The most serious**: under the root view a bucket looks exactly like an ordinary directory, the confirmation only says "Delete folder xxx?", and one misclick would take the whole bucket's data with it. Changed to pass `BucketNotEmpty` through as is — S3 refusing to delete a non-empty bucket is itself a safety property, and the client should not get around it |

---

## 8. Size of the change

| Item | Count |
|---|---|
| Files changed | 43 |
| New S3 source | about 6,083 lines |
| New S3 tests | about 2,832 lines |
| New localisation entries | 129 × 5 languages (en / zh-Hans / zh-Hant / ja / ko) |

Main new files:

```
plugins/VelaShell.Plugin.S3/      protocol implementation + management service + two panels (moved from built-in to a plugin, 2026-08)
src/VelaShell.Core/Models/        S3Settings
src/VelaShell.Infrastructure/S3/  S3FileService, S3ManagementService, S3ClientFactory,
                                  S3ConfigJson, S3Interop, S3CertificateProbe, IS3ClientAccessor
src/VelaShell/ViewModels/S3/      S3ManagerViewModel, S3ObjectInspectorViewModel,
                                  S3ConfigForm, S3FormField
src/VelaShell/Views/              S3ManagerWindow, S3ObjectInspectorWindow
```

---

## 9. Known issues and to-dos

### 9.1 The only failing test is a pre-existing baseline problem

`XshellImportTests.Scan_RealXshellSessions_RecoversAtLeastOnePassword` reads the real Xshell data on this machine.
The same failure was reproduced in a clean worktree on **the unmodified `HEAD`**, so it is unrelated to this change
(this work did not touch `XshellImportService`).

### 9.2 Still to be verified on real servers

Every conclusion above holds on the loopback server. Before going live, a connectivity check against each of these is
recommended:

- **MinIO**: self-signed certificate + path-style addressing + plain HTTP
- **Real AWS S3**: virtual-hosted style + cross-region bucket + versioning + object lock

Above all, verify the three defaults in section 6 — they were chosen deliberately from the protocol documentation, but
can only be verified synthetically.

### 9.3 Build prerequisite

This repository's `global.json` pins `11.0.100-preview.7.26381.103`, and that SDK is not installed on the current
development machine (only 9.0.307 / 10.0.300). This work **did not modify `global.json`**; instead the SDK was installed
side by side into a temporary directory and invoked by its full path. Anyone else needs to install the matching SDK
before building.
