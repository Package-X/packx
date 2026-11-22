# PackX — Master Planning Index

## 1. Purpose

PackX is the permanent user-facing command and package-management layer for PackageX. The packx command remains installed while Engine versions, SDK packages and supporting components can be installed, updated, pinned, verified, repaired or removed independently.

A developer should not need to understand the internal Rust crate graph or manually locate individual writers. They request the required Toolbelt contents and PackX resolves a tested, compatible Toolchain.

## 2. System boundaries

| System | Owns | Must not absorb |
|---|---|---|
| PackX (this repository) | Permanent shim/CLI, Engine inventory, SDK manager, compatibility/catalog lookup, CAS, locks, offline bundles, Toolchain resolution | .px project semantics, per-format writer internals, end-user installer runtime |
| PackageX Engine ([packagex](https://github.com/Package-X/packagex)) | .px parser/linter, resolved project IR, build graph, resource inventory, orchestration and project policy | Global Toolbelt store and target writer implementation |
| PackageX SDK ([sdk](https://github.com/Package-X/sdk)) | Coherent target SDK packages, writers, resource transforms, Electron adapter, PDC and format verification | Global package installation, Engine routing or end-user runtime management |
| PackageX Runtime | End-user installer/application execution and declared application-runtime acquisition | Developer Engine/SDK installation |

## 3. Toolbelt and Toolchain

### Toolbelt

The Toolbelt is the machine-managed installation pool. It contains the permanent PackX shim, one or more Engine versions, target SDK packages and dependencies, package manifests, caches, lock records and ownership/reference metadata.

The Toolbelt may contain packages not used by the currently active project because another project or pinned Toolchain still references them.

### Toolchain

A Toolchain is a resolved, compatible selection for a project/build/target:
- exact Engine version/build;
- exact SDK package Stable version or Nightly commit identity;
- SDK API compatibility declaration;
- exact component versions and full digests;
- requested target formats/capabilities;
- build-host constraints;
- relevant offline/reproducibility/signing policy references.

A Toolchain is not the entire installed Toolbelt. Several Toolchains can coexist.

## 4. Canonical command family

~~~powershell
packx toolbelt list
packx toolbelt install <package-id[@identity]>
packx toolbelt install <package-id> --nightly
packx toolbelt install <package-id> --nightly=a1b2c3d4
packx toolbelt install --required
packx toolbelt install --required --nightly
packx toolbelt install --required --nightly=a1b2c3d4
packx toolbelt verify
packx toolbelt repair
packx toolbelt export --required -o required.pxbelt
packx toolbelt import required.pxbelt
packx toolchain list
packx toolchain inspect
packx toolchain resolve
~~~

The central Engine repository currently contains older packx belt examples. This repository establishes packx toolbelt as canonical public spelling. Before first public CLI implementation, decide whether packx belt is a temporary compatibility alias or removed. They must never be separate managers/state stores.

## 5. Document map

| Document | Focus |
|---|---|
| [Toolbelt and Toolchain](architecture/toolbelt-and-toolchain.md) | Terminology, machine store, multiple Engine/Toolchain coexistence and ownership |
| [Required resolution](architecture/required-resolution.md) | --required, project discovery, dependency closure, minimal install and Nightly selection |
| [Compatibility registry](architecture/compatibility-registry.md) | Signed data, Engine↔SDK/API/target compatibility and privacy |
| [CAS and offline](architecture/cas-and-offline.md) | Immutable content store, locking, atomic extraction, references and .pxbelt |
| [Toolbelt CLI](specs/toolbelt-cli.md) | Commands, flags and user-visible semantics |
| [Download trust](security/package-download-trust.md) | Signatures, hashes, archive safety, mirrors and recovery |
| [Dependency/resolution playbook](implementation/dependency-and-resolution-playbook.md) | Candidate Rust crates, Engine handshake, dependency resolution, download/extract/register sequence and POC milestones |
| [Package-store state/recovery](implementation/package-store-state-and-recovery.md) | Manifest/index/lock schemas, state machine, safe extraction, WAL-style registration, leases and crash testing |

## 6. Implementation approach

The dependency and resolution playbook specifies a concrete synchronous Rust-first implementation candidate (ureq, fs4, tempfile, serde, semver, Minisign verification and safe archive readers), explains why a full dependency solver is not initially required, and defines the Engine requirements handshake that prevents PackX from duplicating .px parsing. The package-store document defines proposed manifest/index/lock data, the download state machine, pre-scan extraction checks, crash-recoverable CAS registration and process leases. These are implementation plans, not existing code or locked dependency versions.

## 7. Installation invariants

1. Install only packages required by the selected Engine/project/target and declared dependency closure.
2. Never install unrelated OS, architecture or format packages.
3. Optional SDK components install only when a requested capability requires them.
4. Resolve/preflight before mutation.
5. Verify package digest and target before registration.
6. Installed content is immutable.
7. Preserve existing pinned Toolchains during upgrades.
8. Support side-by-side Engine and SDK versions.
9. Stable is default; Nightly must be selected explicitly, and an unqualified `--nightly` selects the latest compatible Nightly.
10. Offline mode never accesses the network.

## 8. Release identity

Stable examples:

~~~text
px-sdk-windows-x64@2.4.0
px-stub-windows-x64@1.4.2
~~~

Nightly examples:

~~~text
px-sdk-windows-x64@nightly.a1b2c3d4
px-stub-windows-x64@nightly.a1b2c3d4
~~~

For Nightly, the eight-character commit prefix is the visible identity. Full SHA and internal semantic-version metadata remain in signed package metadata. No Beta channel exists.

## 9. Open design decisions

- Whether project-level Toolchain pins live in packx.lock, a project-local toolchain file, or both.
- Exact root directory names and platform conventions.
- Whether `--required` should demand an explicit .px path when project discovery is ambiguous.
- Whether --nightly=<8hex> applies to every required package built from one source commit or requires per-package resolution when commits differ.
- How Engine upgrades interact with locked SDK packages.
- Exact exit-code allocation, synchronized with the central Engine specification.

These questions do not change the core invariant: only the requested dependency closure is installed.
