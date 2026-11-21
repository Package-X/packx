# PackX

**The permanent, lightweight command-line entry point and manager for PackageX Engines, SDK packages, Toolbelts and Toolchains.**

PackX is the installation and resolution layer around the PackageX Engine. It keeps the user-facing packx command stable while allowing multiple Engine versions and compatible SDK packages to coexist, be verified, pinned, exported and repaired.

> **Project status:** Planning repository. The documents describe intended architecture and CLI behavior; they do not claim that implementation or command set already exists.

## Responsibility boundary

- **PackX** owns the permanent CLI shim, Toolbelt inventory, SDK package acquisition, compatibility lookup, content-addressed storage (CAS), locking, offline bundles, pinning, verification, repair and Toolchain resolution.
- **PackageX Engine** owns .px parsing, project semantics, build planning, resource inventory and build orchestration.
- **PackageX SDK** owns target-specific writers, format adapters, Electron analysis/package preparation, payload optimization and artifact verification.
- **PackageX Runtime** runs installed application/installer behavior on end-user systems. It is not the developer Toolbelt.

## Toolbelt versus Toolchain

- **Toolbelt** is the installed, machine-managed pool of available PackageX Engines, SDK packages and explicitly required supporting tools.
- **Toolchain** is the compatible selection used for one project/build/target: an exact Engine plus required SDK packages/components, target capabilities and pinned hashes.

A machine can have multiple Toolchains. Installing a newer SDK must not silently replace a project's pinned Toolchain.

## Core behavior

1. packx toolbelt install --required resolves what the selected Engine/project needs and installs only missing required packages and dependencies.
2. SDKs are installed as coherent target packages containing multiple compiled pieces, not as individual Rust crates.
3. Compatibility comes from signed, machine-readable metadata, not guessed version arithmetic or usage telemetry.
4. Stable uses semantic versions. Nightly uses an eight-character Git commit identity. There is no Beta channel.
5. --nightly selects the latest published compatible Nightly; --nightly=<8char_commit> selects an exact snapshot. Nightly is never selected automatically.
6. Verified packages are immutable in CAS. Downloads are hash-checked, extracted atomically and reference-counted.
7. Offline .pxbelt export/import supports air-gapped environments.
8. Unrelated OSes, architectures, formats and optional components are not downloaded merely because they exist.

## Planning map

- [Master PackX plan](plan/index.md)
- [Toolbelt and Toolchain architecture](plan/architecture/toolbelt-and-toolchain.md)
- [Required dependency resolution](plan/architecture/required-resolution.md)
- [Compatibility registry](plan/architecture/compatibility-registry.md)
- [CAS, locks and offline bundles](plan/architecture/cas-and-offline.md)
- [Toolbelt CLI specification](plan/specs/toolbelt-cli.md)
- [Package download and trust model](plan/security/package-download-trust.md)
- [Rust dependency and resolution playbook](plan/implementation/dependency-and-resolution-playbook.md)
- [Package-store data model and crash recovery](plan/implementation/package-store-state-and-recovery.md)

## Related repositories

- [PackageX Engine](https://github.com/Package-X/packagex)
- [PackageX SDK](https://github.com/Package-X/sdk)

Command examples are proposed behavior until implemented and tested. Compatibility examples are illustrative, not declarations of currently tested support.
