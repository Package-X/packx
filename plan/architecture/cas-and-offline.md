# CAS, Atomic Installation and Offline Toolbelt Bundles

## 1. Purpose

Content-Addressed Storage (CAS) lets multiple Engine versions and Toolchains reuse identical verified SDK package bytes without duplicate downloads. CAS stores immutable package objects. Toolchain locks and the package index determine ownership.

## 2. Storage concepts

- **Package identity:** SDK package ID plus Stable/Nightly visible identity.
- **Manifest digest:** full digest of canonical package manifest.
- **Artifact digest:** full SHA-256 of each archive/file.
- **CAS object:** extracted immutable package tree or archive keyed by content identity.
- **Owner:** Toolchain/Engine/dependency that references the object.
- **Index:** mapping of package identities to CAS objects and owner references.

A shortened digest prefix may name a filesystem directory, but every object is verified with the full digest.

## 3. Illustrative layout

~~~text
~/.packx/
├── bin/packx
├── state/
│   ├── toolbelt.lock
│   ├── package-index.json
│   └── catalogs/
├── engines/
│   └── <engine-version>/
├── sdk/
│   └── <target-slug>/
│       └── <digest-prefix>_<package-id>/
├── cache/
│   ├── downloads/
│   └── extracted/
└── bundles/
~~~

Actual locations follow platform-native user data/cache directories. Do not hardcode the home path or assume case sensitivity.

## 4. Immutable object creation

1. Resolve signed package manifest.
2. Download archive to a unique temporary path.
3. Verify exact size and SHA-256.
4. Verify package manifest signature/trust chain.
5. Extract into a unique temporary directory with path/link/size/file-count limits.
6. Verify every extracted file against its manifest.
7. Flush required file and directory state.
8. Atomically rename into final CAS location on the same filesystem.
9. Mark immutable/read-only where practical.
10. Update package identity and owner references through atomic index replacement.

Never register incomplete content or extract over an existing object.

## 5. Cross-process locking and index updates

Mutating operations acquire an OS-managed file lock (for example LockFileEx on Windows and flock on supported Unix systems). PID/command metadata exists for diagnostics only; correctness depends on the OS lock, not stale PID cleanup.

The lock protects:
- install/update/repair/remove/prune;
- CAS registration/reference updates;
- package-index updates;
- Toolchain lock changes involving local package state.

Index transaction:
1. Read a consistent snapshot.
2. Apply changes in memory.
3. Validate package/owner/reference invariants.
4. Write a sibling temporary index.
5. Flush it.
6. Atomically replace the index.
7. Use a recovery journal if several filesystem objects must change together.

## 6. Sharing and reference counting

A package can be referenced by multiple Toolchains/Engines. Removing one owner removes only that reference; bytes remain while any owner exists.

Prefer explicit owner sets over an opaque counter because sets can be reconciled after crash recovery. Active process leases also prevent deletion while a package is executing.

## 7. Update and repair

Installed CAS objects are immutable.

- Update downloads a new immutable identity and changes only eligible unpinned references.
- Repair verifies the exact installed object.
- If corrupt, stage a fresh copy of the same identity and replace the CAS mapping atomically.
- Never edit damaged files in place.
- If the exact identity is revoked/unavailable, explain recovery choices rather than silently substituting another version.

## 8. Garbage collection

Prune removes objects only when no Engine/Toolchain owner references them, no active process lease exists, no transaction journal references them and no protected offline/import operation is using them.

Provide --dry-run with package IDs, digests and expected reclaimed bytes. A GC crash must be recoverable and must not invalidate live Toolchains.

## 9. .pxbelt offline bundles

Default offline export includes the exact Engine and SDK packages needed by one selected Toolchain plus transitive dependencies.

~~~powershell
packx toolbelt export --required -o project.pxbelt
packx toolbelt import project.pxbelt
~~~

Bundle manifest includes schema/version, exact Engine identity, exact SDK package/component identities, catalog revision, manifest digests, artifact sizes/hashes, target/dependency closure, signatures/provenance.

An Electron runtime is an end-user application dependency, not normally part of the developer Toolbelt bundle. A separate application/offline export operation may include the exact runtime when required.

Import verifies signed manifest trust and every package artifact before adding anything to CAS. A bundle cannot grant trust to its own arbitrary signing key; trust must chain to configured PackageX release keys or an explicit administrator-approved trust operation.

## 10. Offline and locked semantics

- **Offline:** no network calls; missing exact identity is a hard error with .pxbelt guidance.
- **Locked:** no package identity changes beyond project lock.
- Cached signed catalogs may be used only under freshness/pin policy.
- A missing Nightly commit is never replaced by another Nightly or Stable.
- Report exact missing package identity.

## 11. Acceptance criteria

- [ ] Same package identity is stored once and shareable.
- [ ] Full hashes are verified before registration/use.
- [ ] Index and extraction are atomic/recoverable.
- [ ] Concurrent mutation is serialized by OS lock.
- [ ] Active Toolchain references prevent deletion.
- [ ] Repair stages a fresh immutable object.
- [ ] Prune supports dry-run and respects active-use references.
- [ ] .pxbelt includes exact dependency closure and verifies before import.
- [ ] Offline mode never accesses network.
