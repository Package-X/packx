# Toolbelt Package Store: Data Model, State Transitions and Crash Recovery

## 1. Purpose

The package store turns signed remote SDK/Engine packages into locally usable immutable objects.

The critical invariant is:

**A package is either absent, or fully verified and registered. A partially downloaded or partially extracted package is never a usable Toolbelt package.**

The store must preserve Engine/Toolchain coexistence, make crash recovery deterministic and avoid downloads when a verified identical package is already present.

## 2. Canonical identity layers

| Identifier | Example | Purpose |
|---|---|---|
| Package ID | px-sdk-windows-x64 | Stable product identity |
| Stable version | 2.4.0 | Human-readable immutable release |
| Nightly identity | nightly.a1b2c3d4 | Human-readable commit selector |
| Full source commit | full Git SHA | Exact upstream source identity |
| Manifest digest | full SHA-256 | Identity of signed package metadata |
| Archive digest | full SHA-256 | Identity of downloaded archive bytes |
| File digest | full SHA-256 | Integrity of one extracted file |
| CAS object key | archive/content digest | Physical immutable storage key |
| Toolchain ID | lock digest or stable generated ID | Project's resolved Engine/SDK composition |

A package version is not a content hash. A source commit is not an archive hash. The signed manifest binds these identities together.

## 3. Proposed package manifest

~~~json
{
  "schema": "packagex-package",
  "schema_version": 1,
  "package": {
    "id": "px-sdk-windows-x64",
    "channel": "stable",
    "identity": "2.4.0",
    "version": "2.4.0",
    "target": {
      "platform": "desktop",
      "os": "windows",
      "architecture": "x86_64",
      "abi": "msvc"
    }
  },
  "source": {
    "repository": "Package-X/sdk",
    "commit": "full-git-commit",
    "workflow_run": "run-identity"
  },
  "compatibility": {
    "sdk_api": "1",
    "engine": ">=2.3.0 <2.7.0"
  },
  "dependencies": [
    {
      "package": "px-common-core",
      "range": ">=1.2.0 <2.0.0",
      "required": true
    }
  ],
  "artifacts": [
    {
      "name": "sdk-package.zip",
      "url": "https://releases.example.invalid/sdk-package.zip",
      "size": 12345678,
      "sha256": "full-64-hex-sha256"
    }
  ],
  "files_manifest": {
    "path": "files.json",
    "sha256": "full-64-hex-sha256"
  },
  "license_manifest": "licenses/index.json",
  "sbom": "sbom.spdx.json"
}
~~~

This is a planning example, not a frozen schema. A real schema must define canonical JSON serialization, integer bounds, URI rules, SemVer range grammar, signature envelope and unsupported-field behavior.

### Required validation rules

- Reject unknown major schema versions.
- Preserve unknown non-critical metadata for forward compatibility where safe.
- Reject unknown fields marked critical by schema policy.
- Reject non-normalized package IDs and target identifiers.
- Require full-length SHA-256 values in lowercase canonical hex.
- Require exact size fields for downloadable artifacts.
- Validate URL scheme/origin against catalog and local policy.
- Require a package manifest to bind every artifact to target, version, source commit and channel.
- Never infer channel from a tag substring if signed metadata declares a different channel.

## 4. Local package index

The local index is a cache/ownership database, not a trust source. The signed manifest remains the authority for artifact identity and compatibility.

Conceptual shape:

~~~json
{
  "schema": "packx-package-index",
  "schema_version": 1,
  "generation": 12,
  "packages": {
    "px-sdk-windows-x64@2.4.0": {
      "manifest_sha256": "full-sha256",
      "archive_sha256": "full-sha256",
      "cas_path": "sdk/windows-x64/ab12.../px-sdk-windows-x64",
      "target": "windows-x86_64",
      "verification": "passed",
      "owners": [
        "toolchain:project-a:windows-x64"
      ],
      "installed_at": "timestamp"
    }
  },
  "transactions": []
}
~~~

The index is regenerated/reconciled from package manifests, Toolchain locks and transaction journals when corruption is detected. Do not make an integer refcount the only ownership evidence; a set of owner IDs is easier to reconcile.

The exact package key includes immutable identity. Two different Nightly commits must never map to the same key.

## 5. Project Toolchain lock

Conceptual example:

~~~json
{
  "schema": "packx-toolchain-lock",
  "schema_version": 1,
  "project": {
    "manifest_sha256": "full-sha256",
    "engine": {
      "version": "2.3.0",
      "artifact_sha256": "full-sha256"
    }
  },
  "selection": {
    "channel": "stable",
    "catalog_revision": 42
  },
  "packages": [
    {
      "id": "px-sdk-windows-x64",
      "identity": "2.4.0",
      "manifest_sha256": "full-sha256",
      "archive_sha256": "full-sha256",
      "source_commit": "full-commit"
    }
  ]
}
~~~

Nightly uses identity nightly.a1b2c3d4, with internal semantic version metadata and full commit. The lock stores the full commit, not just the prefix.

Lockfile changes are explicit:
- initial resolve may create the lock;
- ordinary build uses the lock;
- update check does not rewrite it;
- toolchain upgrade intentionally changes it;
- --locked rejects any operation that would change it.

## 6. Download state machine

~~~text
UNRESOLVED
  -> MANIFEST_VERIFIED
  -> DOWNLOAD_PENDING
  -> DOWNLOADING
  -> ARCHIVE_VERIFIED
  -> EXTRACTING
  -> FILES_VERIFIED
  -> CAS_STAGED
  -> CAS_REGISTERED
  -> TOOLCHAIN_ATTACHED
~~~

Failure handling:
- before download: no mutation;
- during download: temporary partial file only;
- archive hash mismatch: delete/quarantine partial; never extract;
- extraction failure: remove staging tree or leave a transaction-journal entry for recovery;
- file hash mismatch: reject entire staged package;
- CAS rename failure: keep index unchanged;
- index update failure after rename: journal identifies unreferenced complete object for safe recovery/reconciliation;
- Toolchain lock write failure: keep verified package registered but do not claim project lock changed; report explicit retry.

Do not collapse these states into one boolean installed flag.

## 7. Download implementation details

With a synchronous HTTP client such as ureq:

1. Construct request from signed manifest URL.
2. Apply TLS/origin/policy configuration.
3. Set connect/read timeouts and response-size limit.
4. Stream response body into a unique file in the download cache.
5. Update SHA-256 incrementally while counting bytes.
6. Abort if declared size is exceeded.
7. Compare final byte count and digest.
8. Flush and close the file.
9. Parse/verify detached signature where the artifact format requires one.
10. Move to archive-verified state.

Never trust:
- ETag;
- Last-Modified;
- Content-Length by itself;
- URL filename;
- hash metadata served by an untrusted mirror.

Resume support is allowed only when the server supports safe byte ranges and the final full archive digest is recomputed over the completed file before acceptance.

## 8. Extraction implementation details

### Pre-scan

Before extraction, enumerate all archive members and validate:
- normalized relative path;
- no absolute path or drive prefix;
- no path traversal;
- no duplicate normalized path;
- no case-folding collision where target filesystems are case-insensitive;
- no unsupported device/FIFO/socket entries;
- no symlink/hardlink escape;
- declared file-count and expanded-size limits;
- per-file limit and compression-ratio threshold;
- allowed permissions.

Archive-aware path validation such as ZIP's enclosed_name() is one check, not the entire security policy. Validate every archive member before extraction, and extract into a private unique staging directory.

### Write pass

- Create a new private staging root with restrictive permissions.
- Create directories/files beneath the staging root only.
- Do not follow symlinks while extracting.
- Create links only after validating target and only when the package schema allows them.
- Stream each file into its destination and hash while writing.
- Set executable permissions only from the signed manifest after file content verification.
- Reject archive entries not represented in the signed files manifest unless a schema-defined metadata entry permits them.

### Final verification

Compare:
- expected path set vs extracted path set;
- expected size vs actual size;
- expected SHA-256 vs actual SHA-256;
- declared mode vs applied mode;
- target executable architecture vs package target.

Only after every member passes can the package reach FILES_VERIFIED.

## 9. Atomic registration protocol

A single rename is not a transaction across both the filesystem and JSON index. PackX needs a small write-ahead journal.

Suggested protocol:
1. Under the Toolbelt mutation lock, create a transaction record with ID, package identity, staging path, intended CAS path and intended owners.
2. Flush the journal.
3. Rename the complete verified staging directory to the final CAS object location.
4. Write a new package-index snapshot to a sibling temporary file.
5. Flush the new index.
6. Atomically replace package-index.json.
7. Mark the transaction committed/remove journal.
8. Release lock.

Crash cases:
- journal exists, final CAS absent: remove or resume staging according to recorded state;
- final CAS exists, index not updated: verify manifest/object and finish index commit, or delete orphan if transaction is invalid;
- index refers to package, object missing: mark index inconsistent and repair from signed source; never load it;
- both index and CAS exist but hashes mismatch: quarantine object and mark package invalid.

The index must never be edited in place. Keep previous index backup or compact journal until recovery policy can prove the transaction completed.

## 10. Cross-process locking

One OS-level lock serializes mutations. The lock file may contain:
- process ID;
- command and package ID;
- process start timestamp;
- diagnostic transaction ID.

Do not infer ownership from a PID alone. The OS lock is authoritative. The lock must be released automatically after process termination.

Reads should consume a complete index snapshot. They should not observe a half-written JSON file.

## 11. Garbage collection and leases

Each package has explicit owners:
- Toolchain lock;
- installed Engine package;
- another SDK package dependency;
- active process lease;
- recovery transaction.

A package is collectable only when all owners, leases and transaction references are absent.

Process leases prevent a build from loading a package while another process removes it. A small per-package shared lock/lease file can be held while the SDK is loaded or worker process is alive. Removal/prune requires an exclusive lease.

Do not delete a package merely because a reference count says zero; reconcile the lock and owner set first.

## 12. Offline bundle format

A .pxbelt bundle is an archive plus a signed root manifest. It contains exactly one selected Toolchain's required Engine/SDK packages and dependency closure unless an explicit different export mode is introduced.

The root manifest contains package identities, hashes, manifests, Engine identity, catalog revision and signatures. Each nested package archive retains its own digest/signature identity.

Import:
1. Open the bundle in read-only mode.
2. Verify bundle signature/trust root.
3. Validate every entry path/size/count.
4. Verify each nested package hash.
5. Verify package manifests/catalog references.
6. Stage every package.
7. Verify extracted file inventories.
8. Register all packages with one recoverable bundle transaction.
9. Mark the imported Toolchain available only when the complete dependency closure is registered.

A partially imported bundle may leave already verified packages in CAS as unowned objects, but it must not create a complete Toolchain record until all required packages are present and verified.

## 13. Recovery scenarios to test

- Process kill at every download/extract/register transition.
- Power loss immediately before and after final CAS rename.
- Disk full during archive write, extraction, index write and journal write.
- Two concurrent installs of same package.
- Two concurrent installs with shared dependency.
- Toolchain removal while SDK is in use.
- Prune started while a worker process is active.
- Index JSON truncated or corrupted.
- CAS object file modified after installation.
- Catalog revoked after package was installed but before new resolve.
- Offline bundle interrupted after one package of several.
- Nightly 8-hex prefix collision.

## 14. Acceptance criteria

- [ ] Local index never becomes the sole source of package authenticity.
- [ ] Manifest binds package identity, channel, commit, target and hashes.
- [ ] Every package transitions through explicit states.
- [ ] Extraction pre-scans archive members and enforces limits.
- [ ] Filesystem registration and index update are crash-recoverable.
- [ ] Toolchain lock records full immutable package identities.
- [ ] Owner sets and process leases protect live packages.
- [ ] .pxbelt import is all-or-nothing at Toolchain availability level.
- [ ] Interrupted work never appears installed until fully verified.
