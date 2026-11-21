# PackageX (PackX) Engineering & Agent Ground Rules

Read this file before modifying specifications or implementation code in this repository.

## 1. Workflow discipline

- **Planning vs. implementation:** Keep planning/specification edits separate from Rust implementation unless explicitly instructed to begin implementation.
- **Plan synchronization:** Changes to CLI commands, package identities, lock formats, compatibility metadata or exit codes update the related plan in the same change.
- **Repository boundaries:** PackX owns package management and Toolchain resolution. It must not absorb Engine project semantics or SDK writer implementations.

## 2. Formatting and code quality

- Follow the formatting convention in the actual language/toolchain configuration.
- Keep modules small with explicit responsibilities.
- Document public APIs and non-obvious invariants.
- Prefer deterministic, typed errors and testable state transitions.
- Avoid dependencies that duplicate functionality already present in the runtime.

## 3. Non-negotiable management invariants

1. **Minimal installation:** Never install unrelated OS, architecture, format or optional components just because they are available in a catalog.
2. **Immutable packages:** Verified CAS content is immutable. Repair replaces an object through staging; it never patches installed files in place.
3. **Side-by-side coexistence:** Multiple Engine versions and Toolchains coexist without silently upgrading another project's pinned selection.
4. **Lock before mutation:** Package-store, Toolchain and reference-index updates use the documented cross-process lock and atomic update protocol.
5. **Verify before use:** Do not execute/load SDK binaries or register packages until signatures, full hashes, target identity and extraction safety pass.
6. **Nightly identity:** Visible Nightly identity derives from the selected Git commit and uses an eight-hex prefix. Internal semantic version remains metadata. There is no Beta channel.
7. **Offline honesty:** Offline/locked operations never access the network or substitute another package identity.
8. **Runtime separation:** Developer Toolbelt SDK packages and end-user application runtimes (including Electron) are separate stores and lifecycle owners.
9. **Explicit updates:** Updating installed packages and changing a project's pinned Toolchain are distinct operations.

## 4. Planning-first implementation order

When implementation begins, establish a working vertical slice:
1. Permanent PackX shim and CLI routing.
2. Toolbelt inventory, lock and atomic index updates.
3. Signed catalog parsing and Engine/SDK compatibility resolution.
4. Required installation for one supported target.
5. CAS download, hash verification, atomic extraction and registration.
6. Toolchain lock and side-by-side Engine routing.
7. Offline .pxbelt export/import.
8. Update, verify, repair, remove and safe garbage collection.
9. Multi-target and Nightly expansion.
