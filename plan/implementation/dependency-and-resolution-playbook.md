# PackX Implementation Map: Rust Dependencies, Resolver and Installation Pipeline

> Research snapshot: 21 November 2026. Dependency versions and APIs are not Cargo pins. Before implementation, re-check releases, MSRV, license, maintenance, security advisories and platform behavior. This plan records concrete candidates and a proof-of-concept path; no PackX implementation is claimed here.

## 1. Design principle

PackX should be a small, dependable package manager, not another package ecosystem that downloads the entire internet.

Its job is to:
1. identify the intended Engine/project;
2. ask the Engine what SDK capabilities and packages are required;
3. resolve only a compatible dependency closure;
4. verify signed catalog and package metadata;
5. fetch exact immutable artifacts;
6. safely extract them into CAS;
7. atomically register them;
8. create or verify a project Toolchain lock;
9. keep existing Engine/Toolchain versions working side by side.

The permanent packx shim stays small. Network transfer, CAS, manifest verification and Toolchain resolution can be reusable Rust libraries behind the CLI.

## 2. Candidate Rust dependency stack

The goal is to start with a small synchronous dependency set. PackX is primarily a package manager and command router; it does not need a large async runtime just to download a few SDK packages.

| Concern | Candidate | How PackX uses it | Notes / boundary |
|---|---|---|---|
| CLI parsing | clap | Parse toolbelt, toolchain, --required, --nightly, --offline, --locked, --project | Keep command parsing in the shim/CLI, not in Engine or SDK |
| HTTP | ureq | Blocking HTTPS GET, response streaming, proxy and TLS handling | Low-overhead synchronous API; choose TLS backend deliberately |
| JSON | serde + serde_json | Catalog, manifests, indexes, lockfiles and machine-readable Engine requirements | Validate schemas/unknown critical fields rather than blindly deserializing |
| SemVer | semver | Parse Engine/SDK versions and test version requirements | Version ranges alone never establish feature compatibility |
| File lock | fs4 or OS-specific audited wrappers | Cross-process lock for Toolbelt mutation | Verify behavior on Windows/network filesystems; lock file contents are diagnostics, OS lock provides exclusion |
| Temporary files | tempfile | Unique download/staging directories and cleanup guards | Final atomic rename must occur on the same filesystem |
| ZIP | zip | Read ZIP-based SDK archives and validate member names | Use enclosed_name; still add duplicate/path-case/size limits |
| TAR | tar | Read TAR members where upstream package uses TAR | Never blindly unpack untrusted input without prevalidation and bounds |
| Minisign verification | minisign-verify | Verify detached signatures on catalog/package manifests | Zero-dependency verifier, streaming for large files; use pinned trusted public keys |
| Minisign signing | minisign or external protected signer | Sign catalogs/release metadata in protected release workflow | PackX client should normally only verify; do not store catalog private key in user Toolbelt |
| SHA-256 | Existing PackageX-approved crypto crate | Verify full package/archive and extracted-file hashes | Use one canonical digest implementation across Engine/SDK/PackX where practical |
| Compression | zstd where bundle format requires it | Decompress .pxbelt or compressed catalog/package bundles | Bound expanded size and memory |
| Dependency solver | Small deterministic DAG resolver initially; pubgrub if ranges/conflicts require | Resolve SDK package + component dependency graph | Don't introduce a full solver until package graph actually needs version solving |
| Console output | indicatif optional | Progress display for large downloads | Optional UX helper; machine-readable mode must not depend on it |

Upstream documentation:
- [ureq](https://docs.rs/ureq/latest/ureq/): synchronous/blocking HTTP API, proxy/TLS support and low-overhead design.
- [fs4](https://docs.rs/fs4/latest/fs4/): cross-platform file locking; current crate also exposes file allocation/space information.
- [tempfile](https://docs.rs/tempfile/latest/tempfile/): temporary files and directories with RAII cleanup.
- [zip](https://docs.rs/zip/latest/zip/): use ZipFile::enclosed_name() to reject archive paths escaping the extraction root.
- [minisign-verify](https://docs.rs/minisign-verify/latest/minisign_verify/): verifies standard/pre-hashed Minisign signatures and supports streaming verification. Its streaming verifier requires a pre-hashed signature; use a matching release-signing mode and add a test vector to prove signer/verifier compatibility.
- [minisign](https://docs.rs/minisign/latest/minisign/): pure-Rust Minisign signing/verification; signing belongs in a protected release environment.
- [semver::VersionReq](https://docs.rs/semver/latest/semver/struct.VersionReq.html): range parsing and matching.
- [pubgrub](https://docs.rs/pubgrub/latest/pubgrub/): full version solver if future catalog dependencies need conflict explanations.

### 2.1 Why ureq instead of Tokio/Reqwest by default?

The Toolbelt's core workload is bounded metadata fetch plus file download, verification and atomic extraction. It is not inherently a long-lived high-concurrency server.

ureq provides a blocking, low-overhead Rust HTTP client and avoids requiring a global async runtime for this path. The install pipeline may still parallelize a small number of independent package downloads using bounded worker threads.

Use an async client only if profiling or a real concurrent-download requirement proves that synchronous I/O is a bottleneck. Do not add Tokio just because another library uses it internally.

### 2.2 Why fs4 or native file locks?

A PID written into toolbelt.lock is not a lock. A process can crash and leave stale metadata.

Use an operating-system lock primitive through a maintained cross-platform library:
- Windows: byte-range/advisory lock backed by supported Windows file-locking behavior.
- Unix: advisory flock/equivalent supported by the crate.

The lock file can contain owner metadata for a waiting process to display, but kernel lock acquisition/release is the synchronization mechanism. Verify the selected crate's semantics on every supported filesystem; network shares and unusual filesystems may need to be declared unsupported or use a dedicated strategy.

### 2.3 Why Minisign for catalogs?

PackX needs a simple asymmetric authenticity mechanism for small JSON catalogs and package manifests. Minisign is a small detached-signature format with Rust verification support.

Suggested trust layout:
- PackX binaries contain a small initial set of trusted PackageX public keys.
- Catalog JSON is canonicalized and signed as a detached Minisign payload.
- PackX verifies the signature before trusting catalog URLs, versions or hashes.
- Each selected SDK package manifest is separately signed or authenticated by the signed catalog digest.
- Artifact bytes are checked against full SHA-256 digests in the authenticated manifest.
- A future key rotation publishes a signed trust/key-transition document verified by an existing trusted key.

The PackX client needs public keys only. Private catalog/release signing keys must never ship with the Toolbelt or sit in a user-writable package directory.

Minisign verification is one candidate, not an irrevocable decision. The signature envelope, key rotation, revocation and canonical JSON rules must be frozen before first stable package publication.

---

## 3. Canonical data objects

Keep these roles separate.

### 3.1 Catalog

A small signed index that locates package manifests and latest/recommended pointers. It should not need to contain the whole SDK component graph for every platform.

### 3.2 Package manifest

A signed description of one immutable SDK package archive:
- package ID and visible identity;
- channel (Stable or Nightly);
- target platform/architecture/OS;
- full source commit;
- internal semantic version metadata;
- SDK API compatibility;
- Engine compatibility;
- required components;
- artifact URLs, sizes and SHA-256 digests;
- per-file inventory or inventory-manifest digest;
- license/SBOM/provenance locations;
- release key identity.

### 3.3 Installed package record

A local index record linking:
- package identity;
- signed manifest digest;
- CAS object path;
- target identity;
- install time;
- verification result;
- Toolchain owners/leases.

This is local state, not a replacement for the signed manifest.

### 3.4 Project Toolchain lock

A project-level lockfile records the exact package/Engine composition selected for repeatable builds.

It should contain immutable package identities and full hashes. It must not lock only a moving latest alias or an eight-character Nightly prefix without the corresponding full commit and digest.

---

## 4. Required resolution: concrete call sequence

The public command:

~~~powershell
packx toolbelt install --required
~~~

should run this pipeline. The key rule is that PackX does not parse the entire .px format a second time. The Engine is the authority for project requirements.

### Step 1 — Find the project

1. Accept --project <path> explicitly.
2. Otherwise discover a project only if the current directory has one unambiguous .px file according to Engine discovery rules.
3. If there are multiple possible projects, stop and request a path.
4. If no project is needed for a package-specific operation, require package/target arguments.

No current-host inference: being on Windows does not mean the developer wants a Windows output package.

### Step 2 — Bootstrap the Engine

PackX needs a small amount of information before it can ask the Engine for detailed SDK requirements:
- Engine version from the project pin;
- possibly one minimal bootstrap Engine if the project does not pin one;
- package catalog trust roots and compatibility endpoint.

If the exact Engine is missing:
1. fetch the signed Engine release catalog;
2. verify selected Engine release manifest;
3. download and hash-check the Engine artifact;
4. install it in its immutable Engine directory;
5. invoke the Engine in a read-only requirements mode.

Do not install an entire SDK package just to discover which SDK is required.

### Step 3 — Ask the Engine for the requirement graph

Proposed Engine machine interface:

~~~powershell
packx-core requirements --project .\package.px --format json
~~~

This is a proposal to coordinate with the Engine CLI spec, not an existing command.

The response should include:
- Engine version/build ID;
- SDK API range;
- target list;
- requested installer formats;
- exact capabilities needed per target;
- required SDK package IDs or capability selectors;
- optional features requested by the project;
- host restrictions;
- any project Toolchain pins;
- deterministic resource/manifest identity.

PackX asks the Engine rather than implementing its own parser for .px, assembly sections, OS step resolution or feature semantics.

### Step 4 — Query the compatibility registry

- Fetch the current signed catalog/compatibility index.
- Verify signature and freshness before using package URLs.
- Filter entries by Engine API/Engine compatibility.
- Filter by exact target OS and architecture.
- Filter by required output formats and writer operations.
- Apply explicit pins and Stable/Nightly policy.
- Reject revoked packages or incompatible component API versions.

The catalog query should not upload project source or file lists. Send only the metadata necessary to select package IDs/versions when a remote query is needed; preferably download a small static signed catalog and resolve locally.

### Step 5 — Resolve dependency closure

Build a directed graph:
- root nodes are Engine-requested SDK package/capability requirements;
- package manifests add required dependencies;
- optional dependencies are included only when the Engine requirement graph explicitly names them;
- shared dependencies are deduplicated by immutable package identity;
- cycles are rejected;
- incompatible version/API/target constraints fail before download.

Initially, a deterministic topological resolver is enough if SDK package dependencies are pinned exactly and the graph is acyclic. Introduce PubGrub only if package-level version ranges create real conflicts that require a solver. A full solver is not required merely because manifests contain semantic versions.

### Step 6 — Create a human-readable preflight plan

Before locking or mutating:
- print the selected Engine and compatibility range;
- show required SDK package IDs and visible identities;
- show which packages are already installed and verified;
- show which packages are missing and will be downloaded;
- show optional components skipped and why;
- show download sizes and final disk-space estimate;
- show selected channel and catalog revision;
- show exact reasons for every root/dependency selection.

Example:

~~~text
Toolbelt resolution
  Engine: 2.3.0
  Project: ./package.px
  Target: windows / x86_64
  Output: MSI

  Required SDK package:
    px-sdk-windows-x64@2.4.0       18.2 MiB  [download]

  Reused:
    px-common-pdc@1.2.0             1.1 MiB  [verified]

  Skipped:
    macOS SDK                       target not requested
    Linux SDK                       target not requested
    MSIX writer                     format not requested
    Electron adapter                no Electron input

  Total download: 18.2 MiB
  Compatibility: supported
~~~

--dry-run should display this plan without changing CAS/index/lock state.

### Step 7 — Lock and re-check

Acquire the cross-process Toolbelt lock only after the plan is understood. Re-read current package index and Toolchain lock while holding the lock. Another process may have installed the same package between preflight and lock acquisition.

If the package is now present and verifies, reuse it and skip the download. Do not download twice because the initial preflight snapshot was stale.

### Step 8 — Download exact immutable artifacts

For each missing package:
1. Resolve the exact archive URL from authenticated package manifest.
2. Enforce HTTPS and origin allowlists.
3. Stream to unique temporary download file with bounded memory.
4. Validate HTTP status, expected byte count and response truncation.
5. Compute full SHA-256 while streaming.
6. Compare size/hash to signed manifest.
7. Validate archive signature or signed manifest chain.
8. Mark corrupt temporary data for deletion/quarantine.
9. Never treat ETag or Last-Modified as an integrity hash.

Resume support is allowed only when the server supports safe byte ranges and the final full archive hash is recomputed from byte zero after completion. A resumed partial file is never trusted on its own.

### Step 9 — Safe extraction

Before writing any file:
- normalize path separators;
- reject absolute paths and drive prefixes;
- reject traversal outside package root;
- reject NUL bytes, device nodes and unsupported special files;
- reject duplicate conflicting paths;
- check case-folding collisions for Windows-compatible package targets;
- reject symlinks escaping package root;
- enforce maximum entry count, per-file bytes, total expanded bytes and compression-ratio limits;
- validate expected permissions and executable bits.

Use archive-aware path validation such as ZIP's enclosed_name() as one check, not the entire security policy. Validate every archive member before extraction, and extract into a private unique staging directory.

After extraction:
- verify every manifest file hash/size;
- reject undeclared files unless explicitly allowed by manifest;
- verify executable architecture/format;
- verify the SDK package entry and required components can be discovered;
- write a completion marker only after all checks pass.

### Step 10 — Atomic CAS registration

1. Flush file contents/metadata as supported.
2. Rename the staging directory into the final digest-keyed CAS object path on the same filesystem.
3. If an identical immutable object already exists, verify it and discard the redundant stage.
4. Atomically update the package index/owner set.
5. Update Toolchain lock only when explicitly allowed by command policy.
6. Mark the package usable only after index transaction completes.
7. Release the Toolbelt lock.

The package directory is read-only/immutable after registration where practical. Future repair creates a new staging object and atomically redirects the index; it does not patch the existing object in place.

### Step 11 — Final verification and report

After installation, PackX asks the SDK entry artifact to provide its capability handshake and confirms:
- loaded package ID/version/source identity match the manifest;
- SDK API range matches Engine;
- required targets/formats/operations are present;
- each component's digest matches;
- package metadata and local CAS record agree.

Only then report installed and verified.

---

## 5. Stable and Nightly selection algorithm

Stable is the default channel. Nightly is explicit. Beta does not exist.

### CLI forms

~~~powershell
packx toolbelt install --required
packx toolbelt install --required --nightly
packx toolbelt install --required --nightly=a1b2c3d4
~~~

### Meaning

- No Nightly flag: select Stable packages that satisfy pins/compatibility.
- --nightly: use latest compatible Nightly snapshot available in the signed catalog.
- --nightly=<8hex>: resolve the exact selected commit prefix. The package manifest contains the full repository commit and exact archive digest.
- Internal semantic version stays in metadata and can appear in verbose information, but concise list/install/download identity is nightly.<8hex>.
- No fallback from a missing Nightly prefix to another Nightly or Stable.
- No silent mix of different Nightly source snapshots unless the catalog explicitly declares and tests the combination.

### Prefix collision

Eight hexadecimal characters are a human-readable selector, not a cryptographic identity. The catalog must ensure the prefix maps to one full commit within the package namespace. If two commits share a prefix for the same component/package, fail closed and require an explicit extended selector rule rather than choose arbitrarily.

### Latest pointer versus lock

latest and --nightly are resolution selectors, not durable locks. The project lock records:
- package ID;
- channel;
- visible identity;
- full source commit for Nightly;
- semantic package/component metadata version;
- package manifest digest;
- archive and extracted file hashes;
- catalog revision/key identity.

A rebuild uses the lock, not the moving latest pointer.

---

## 6. Toolbelt lock and Toolchain lock are different

### Toolbelt lock

A local cross-process mutation lock protects package-store consistency. It is held briefly during install/update/repair/remove/prune/index writes. It is not a project's semantic dependency lock.

### Toolchain lock

A project file (packx.lock or the eventual selected filename) records exact Engine/SDK selections. It should be checked into source control for reproducible CI when the project policy requests it.

### Required behavior

- Multiple Toolchains may refer to one CAS package.
- Removing one Toolchain removes one owner reference.
- The package remains while another Toolchain uses it.
- Running toolbelt update must not silently rewrite pinned project Toolchains.
- toolchain upgrade is the intentional pin-changing operation.
- An Engine version's supported SDK range does not force every project to install the highest compatible SDK. It constrains the resolver to an allowed set.

---

## 7. Update, repair and garbage-collection algorithms

### update --check

- Fetch/verify catalog.
- Compare latest compatible candidates against installed unpinned selections.
- Show changes and compatibility.
- Do not acquire the mutation lock or alter files beyond allowed metadata cache.

### update --required

- Resolve current project requirements.
- Preserve exact project pins.
- Update only eligible unpinned package references according to Stable/Nightly policy.
- Download new immutable objects; never overwrite old package bytes.
- Keep old objects while pinned/used.

### repair

- Verify the exact identity currently referenced.
- If content missing/corrupt, re-fetch that exact identity.
- Verify signature, manifest, archive and extracted inventory.
- Stage and atomically replace only the CAS mapping.
- If exact identity is revoked/unavailable, stop and explain; don't silently install a newer one.

### prune --dry-run

- Derive live owner set from all Engine/Toolchain lock records.
- Include active process leases and pending transaction journals.
- List only objects with no live owners/leases/journals.
- Show exact reclaimable size.
- Delete nothing.

### prune

Repeat all checks under mutation lock, re-check references, then delete candidate immutable objects and update index transactionally.

---

## 8. Proof-of-concept implementation sequence

### POC 1 — Catalog verification

- Build a static sample catalog.
- Sign it with a temporary Minisign release test key.
- Verify it from PackX using minisign-verify.
- Reject wrong key, edited JSON, truncated signature and stale revision.
- Confirm no secret key is present in PackX client files.

### POC 2 — Download and archive sandbox

- Serve a local signed test package over HTTPS.
- Stream download through ureq.
- Verify archive SHA-256.
- Test ZIP/TAR path traversal, duplicate path, symlink escape, huge expanded-size and file-count cases.
- Confirm no file is written outside package staging root.

### POC 3 — Lock and CAS

- Run two processes installing the same package.
- Confirm second waits and reuses the first completed object.
- Kill one process mid-download and mid-extraction.
- Confirm OS lock releases, incomplete stage is not registered, and recovery cleans/reuses safe state.
- Simulate disk-full and rename failure.

### POC 4 — Engine requirements protocol

- Ask a real Engine build to emit structured requirements.
- Test Engine 2.3, 2.4 and a deliberately incompatible SDK API.
- Confirm PackX does not parse .px project semantics itself.
- Confirm Windows-only MSI project on Linux host resolves only the Windows target package if cross-host capability is declared.

### POC 5 — Minimal install

- Project requests only Windows x64 MSI.
- Install one SDK target package and dependencies.
- Confirm macOS/Linux/MSIX/Electron packages are not fetched.
- Run the SDK capability handshake and perform one minimal build.
- Inspect actual CAS contents and package index.

### POC 6 — Toolchain locking and Nightly

- Install two Engine versions side by side.
- Pin two projects to different SDK versions.
- Install a Nightly package by latest selector, then by exact 8-hex prefix.
- Confirm same package ID resolves to the full commit recorded in lock.
- Confirm missing prefix never falls back to Stable.
- Update one project and verify the other lock remains unchanged.

### POC 7 — Offline bundle

- Export one project's exact required Toolchain closure.
- Transfer to a clean disconnected host.
- Import and verify every archive/manifest.
- Build offline.
- Remove one package and verify the command fails with precise import guidance.

---

## 9. Version pinning and maintenance policy

Every direct dependency proposal includes:
- crate/package version;
- upstream repository and exact source tag/commit;
- license/SPDX ID;
- MSRV and target support;
- enabled Cargo features;
- security advisories checked;
- maintenance status and recent release activity;
- reason for use and fallback.

Do not blindly use latest in an implementation lockfile. Cargo.lock records exact resolved transitive versions. Renovation/updates must run package-manager tests and compatibility conformance before promotion.

Minimize default features:
- ureq: select one TLS backend intentionally.
- fs4: start with synchronous locking features; do not pull async runtimes just for file locks.
- zip/tar: enable only required codecs and inspect transitive native dependencies.
- minisign-verify: client side only; signing remains in protected release automation.
- pubgrub: optional until a real range/conflict case demonstrates need.

## 10. Security acceptance criteria

- [ ] Catalog signature is checked before trusting package URLs/hashes.
- [ ] Package manifest digest/signature matches signed catalog reference.
- [ ] Full archive SHA-256 is recomputed after download/resume.
- [ ] Archive extraction rejects traversal, links, path collisions and resource exhaustion.
- [ ] CAS object is not registered before all extracted hashes pass.
- [ ] Toolbelt mutation lock is OS-backed and cross-process.
- [ ] Toolchain references prevent garbage collection.
- [ ] Repair keeps exact identity or fails; no automatic version substitution.
- [ ] Nightly selector is full-commit verified and does not fall back.
- [ ] Offline mode performs no network access.
- [ ] Project source/content is not uploaded for compatibility lookup.
- [ ] Private signing keys and download credentials are absent from package metadata/logs.
