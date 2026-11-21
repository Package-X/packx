# Required SDK Resolution and Minimal Installation

## 1. User-facing objective

The normal setup command is intentionally simple:

~~~powershell
packx toolbelt install --required
~~~

PackX performs dependency analysis and installs only what the selected PackageX Engine/project needs. Developers should not have to select every writer crate or target component manually.

## 2. Meaning of required

Required means the smallest complete, compatible set of packages and dependencies needed to satisfy the selected Engine, project, requested output targets and explicitly enabled build features.

It does not mean every SDK in the repository, every OS, every architecture, every writer format, every optional tool or every published package.

If a project targets only Windows x64 MSI, PackX must not install macOS, Linux, ARM64, MSIX, Electron or unrelated support unless a declared dependency requires it.

## 3. Resolution inputs

PackX considers:
1. selected project path and .px targets;
2. Engine version pin or explicit Engine selection;
3. Engine SDK API and package compatibility requirements;
4. required output formats and features;
5. target OS/architecture and build-host restrictions;
6. project Toolchain lock and package pins;
7. already-installed verified packages;
8. channel selection (Stable default; Nightly only by explicit request);
9. offline/locked policy and trusted catalog revision.

PackX must not infer project targets from the host OS. A Linux host can build Windows output when the SDK declares that capability.

## 4. Resolution pipeline

### Stage 1 — Discover project context

- Use explicit --project <path> when supplied.
- Otherwise discover an unambiguous .px project in the current context according to Engine path rules.
- If multiple projects are plausible, stop and request a path. Never guess.
- A package-oriented command with no project context must name package and target explicitly.

### Stage 2 — Identify Engine

- Read the project Engine pin or explicit override.
- If no pin exists, use the Engine's documented default and show the selected version.
- Verify the Engine binary before use.
- Resolve the Engine release independently from SDK package resolution if absent.

### Stage 3 — Ask Engine for requirements

PackX does not reimplement .px semantics. It asks the selected Engine for a machine-readable requirement description:
- target tuples;
- requested formats;
- SDK API range;
- required SDK package IDs/capabilities;
- feature-dependent optional components;
- host restrictions;
- exact project/Toolchain pins.

The Engine remains authoritative for project meaning.

### Stage 4 — Resolve compatibility/dependency closure

- Query signed Compatibility Registry/catalog.
- Find SDK package candidates compatible with Engine API/range and requested targets.
- Resolve package/component dependencies.
- Reject cycles, missing dependencies, conflicting pins and revoked identities.
- Select the minimum complete package set.
- Reuse already-installed verified identities.
- Prefer exact lock/pin over latest catalog pointer.

### Stage 5 — Preflight plan

Before mutation, prepare a plan containing:
- selected Engine;
- required SDK packages;
- optional packages skipped and why;
- installed packages reused;
- packages to download;
- dependency closure;
- total download size where known;
- disk-space requirement;
- catalog revision and package identities;
- reason each package is required.

Preview is machine-readable and human-readable and must not mutate package state.

### Stage 6 — Lock and re-check

- Acquire cross-process Toolbelt lock.
- Refresh installed package/index state.
- Re-check planned identities to avoid duplicate downloads after another process completes the same install.
- Confirm catalog/pin identity has not changed unexpectedly.
- Revalidate disk space and offline constraints.

### Stage 7 — Download, verify and stage

- Download only missing packages and dependencies.
- Stream to temporary storage.
- Verify signed package manifest, archive hash, size and target identity.
- Extract with path-safety limits to temporary CAS directory.
- Verify extracted files against manifest.
- Never load partially extracted components.

### Stage 8 — Register atomically

- Move verified immutable content into CAS.
- Update package index and ownership references atomically.
- Update the project Toolchain lock only under explicit lock policy.
- Never replace an existing pinned Toolchain without user action.
- Mark available only after registration completes.

### Stage 9 — Report

Show installed, reused, skipped and unresolved packages; selected Engine/SDK compatibility; Stable/Nightly identity; and resulting Toolchain identity.

## 5. Nightly selection

Stable is the default.

~~~powershell
packx toolbelt install --required
packx toolbelt install --required --nightly
packx toolbelt install --required --nightly=a1b2c3d4
~~~

Rules:
- --nightly opts into latest published compatible Nightly.
- --nightly=<8hex> resolves the exact commit prefix.
- Full commit SHA is verified and stored.
- Eight-character prefix identifies one immutable package snapshot.
- Missing/ambiguous prefix is a hard error. Never fall back to another Nightly or Stable.
- Internal semantic version is metadata, not primary visible identity.
- There is no Beta channel.
- Existing exact project pins take precedence unless explicitly changed.

If selected source commit does not produce all required packages, report the unsatisfied packages. Do not mix unrelated nightly commits unless catalog compatibility explicitly permits that combination.

## 6. Multiple SDKs and targets

A project requiring multiple targets installs the union of their package closures. Shared package identities are deduplicated. Do not install optional target support merely because it exists in the catalog.

## 7. Update and upgrade separation

- update --check: report available compatible updates; no mutation.
- update --required: update eligible unpinned packages needed by selected project.
- toolchain upgrade: intentionally choose a new compatible Engine/SDK composition and update project lock.
- pin/unpin: manage exact immutable package identity.

A normal package update must not rewrite a project's pinned Toolchain.

## 8. Failure behavior

- Ambiguous project: stop and ask for explicit project path.
- No compatible SDK: show Engine range, target and missing capability.
- Offline missing package: fail with exact identity and .pxbelt guidance.
- Hash/signature mismatch: quarantine temporary download and fail closed.
- Insufficient disk: fail before extraction/registration.
- Lock timeout: show owner process details.
- Interrupted download: preserve/remove temporary state safely; never register it.
- Interrupted registration: recover journal/index before retry.
- Stale/revoked catalog: stop and explain refresh/recovery policy.

## 9. Acceptance criteria

- [ ] install --required installs exact dependency closure only.
- [ ] Engine defines project requirements; PackX does not duplicate parser policy.
- [ ] Host OS does not determine project target.
- [ ] Verified packages are reused.
- [ ] Optional components remain absent unless requested.
- [ ] Resolution plan is prepared before package mutation.
- [ ] Concurrent installs re-check state under lock.
- [ ] Nightly exact prefix never falls back.
- [ ] Project Toolchain pins are not silently rewritten.
- [ ] Offline mode never accesses network.
