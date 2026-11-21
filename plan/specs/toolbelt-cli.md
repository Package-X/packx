# packx toolbelt and packx toolchain CLI Specification

## 1. Command hierarchy

The canonical package-management namespace is packx toolbelt. Project-specific compatible selections use packx toolchain.

~~~text
packx
├── toolbelt
│   ├── list
│   ├── search <query>
│   ├── info <package-id[@identity]>
│   ├── install --required [--nightly[=<8hex_commit>]]
│   ├── update [--check] [--required]
│   ├── pin <package-id>@<identity>
│   ├── unpin <package-id>
│   ├── remove <package-id[@identity]>
│   ├── verify [<package-id>]
│   ├── repair [<package-id>]
│   ├── prune [--dry-run]
│   ├── export [--required] -o <file>.pxbelt
│   └── import <file>.pxbelt
└── toolchain
    ├── list
    ├── inspect [--project <path>]
    ├── resolve [--project <path>] [--nightly[=<8hex_commit>]]
    ├── lock [--project <path>]
    └── verify [--project <path>]
~~~

This is a proposed command surface. Freeze it against the Engine CLI specification before implementation.

## 2. Required installation

~~~powershell
packx toolbelt install --required
~~~

Behavior:
- Locate intended .px project context.
- Determine Engine pin and project target/feature requirements.
- Read Engine SDK API/compatibility declarations.
- Query signed Compatibility Registry/catalog.
- Resolve only required SDK packages and dependencies.
- Reuse verified packages already installed.
- Download, verify and atomically register missing packages.
- Persist exact Toolchain selection according to lock policy.
- Display installed, reused, skipped and unavailable packages with reasons.

Do not download unrelated OS/architecture/format packages, debug symbols or optional components. If discovery is ambiguous, require --project <path> rather than guessing.

## 3. Nightly selection

Stable is default. Nightly is opt-in.

~~~powershell
packx toolbelt install --required
packx toolbelt install --required --nightly
packx toolbelt install --required --nightly=a1b2c3d4
~~~

Rules:
- --nightly without a value selects latest published compatible Nightly, subject to coherent Toolchain policy.
- --nightly=<8hex_commit> selects the exact snapshot.
- Prefix is exactly eight hex characters; lookup is case-insensitive and display normalized lowercase.
- Full commit SHA is stored and checked against selected workflow source.
- Missing/ambiguous prefix fails. Never fall back to another Nightly or Stable.
- Visible identity is nightly.a1b2c3d4; internal semantic version is metadata.
- No Beta switch, alias or tag exists.
- Explicit project pins take precedence unless explicitly changed.

## 4. Listing and inspection

list shows package ID, target, visible identity, channel, verification state and owner references.

For Nightly, concise output shows commit identity rather than internal version:

~~~text
px-sdk-windows-x64
  Identity: nightly.a1b2c3d4
  Internal version: 2.5.0-dev.18
  Full commit: a1b2c3d4...
  Target: windows / x86_64
  Verification: passed
  Used by: Toolchain 7d...
~~~

info shows compatibility ranges, component inventory, artifact sizes/digests, provenance and catalog status.

## 5. Pinning

~~~powershell
packx toolbelt pin px-sdk-windows-x64@2.4.0
packx toolbelt pin px-sdk-windows-x64@nightly.a1b2c3d4
packx toolbelt unpin px-sdk-windows-x64
~~~

Pinning locks immutable identity. Unpinning allows future resolution under selected channel policy; it does not immediately replace packages unless combined with resolve/update.

## 6. Update behavior

~~~powershell
packx toolbelt update --check
packx toolbelt update --required
~~~

- update --check reports compatible updates without mutation.
- update --required updates eligible unpinned packages required by selected project/toolchain.
- It does not advance pinned Stable versions or Nightly commits.
- toolchain upgrade is the explicit operation to choose a new Engine/SDK composition and rewrite the project lock.

## 7. Verify, repair and remove

- verify checks signatures, manifest, artifact hashes, components, target identity and index references.
- repair re-fetches exact pinned identity and replaces corrupt content through staging. If unavailable/revoked, stop with recovery guidance.
- remove releases one owner reference; delete bytes only when no references remain.
- prune --dry-run reports candidates and expected reclaimed bytes.
- prune removes unreferenced objects respecting active-use leases and journals.

## 8. Offline export/import

~~~powershell
packx toolbelt export --required -o project-toolchain.pxbelt
packx toolbelt import project-toolchain.pxbelt
~~~

export --required includes the selected project's Engine, exact required SDK packages and full dependency closure, not the entire machine Toolbelt. A full-machine export, if ever added, is an explicit separate option.

## 9. Diagnostics and exit codes

Reuse the central Engine/PackageX exit-code catalog. Proposed stable error conditions include:
- no/ambiguous project;
- no compatible SDK package;
- Nightly prefix missing/ambiguous;
- invalid/stale catalog;
- artifact hash mismatch;
- unsafe archive;
- lock timeout;
- offline dependency missing;
- insufficient disk;
- package corruption/repair failure.

Do not allocate ad-hoc numeric exit codes before synchronizing central error specifications.

## 10. Legacy packx belt spelling

The central Engine plan contains older packx belt examples. Canonical command is packx toolbelt. Before first public release, choose a temporary warning alias or remove the old spelling. Both must map to one implementation and state store.

## 11. Acceptance criteria

- [ ] install --required installs only required dependency closure.
- [ ] Stable is default and Nightly is explicit.
- [ ] --nightly=<8hex> resolves exactly and never falls back.
- [ ] Beta is absent.
- [ ] Nightly concise display uses commit prefix; verbose metadata includes full SHA/internal version.
- [ ] Verify/repair/remove/prune/export/import follow immutability/reference rules.
- [ ] Toolchain commands show project-specific selection, not the whole machine pool.
