# Toolbelt and Toolchain Architecture

## 1. Why two concepts are necessary

PackageX needs a stable command that can manage multiple Engine and SDK versions without forcing one global version on every project.

The **Toolbelt** is the machine's installed pool. A **Toolchain** is one compatible selection drawn from that pool for a particular project/build/target.

Conflating them would make one project's update capable of unexpectedly changing another project's build.

## 2. Toolbelt ownership

PackX owns the permanent shim and the local managed package store:
- one stable packx entry point;
- side-by-side PackageX Engine releases;
- SDK target packages and their internal components;
- shared supporting tools explicitly required by the catalog;
- package manifests, digests, ownership references and cache metadata.

The Toolbelt is not a requirement to download every published SDK package. It contains only requested packages or packages retained by a Toolchain.

## 3. Toolchain ownership

A Toolchain resolves:
- Engine release/build;
- SDK package for each required target;
- exact component composition inside each SDK package;
- SDK API compatibility;
- output-format capabilities;
- host restrictions;
- full artifact digests;
- Stable/Nightly identity;
- project pin/lock state.

The Toolchain must be reproducible. Resolving the same lock against the same trusted catalog should select identical immutable identities or fail clearly if unavailable/revoked.

## 4. Illustrative storage layout

~~~text
~/.packx/
├── bin/
│   └── packx
├── state/
│   ├── toolbelt.lock
│   ├── package-index.json
│   ├── engine-index.json
│   └── catalogs/
├── engines/
│   ├── 2.3.0/
│   └── 2.6.0/
├── sdk/
│   ├── windows-x64/
│   │   └── <package-digest>/
│   ├── linux-x64/
│   │   └── <package-digest>/
│   └── macos-arm64/
│       └── <package-digest>/
├── tools/
└── cache/
~~~

Actual locations follow platform-native user data/cache conventions. The tilde above is illustrative, not a literal directory.

Project-local Toolchain record:

~~~text
project/
├── package.px
└── packx.lock
~~~

## 5. Side-by-side coexistence

A machine may have Engine 2.3 and 2.6 installed, with separate projects pinning different SDK package versions. PackX routes by project Engine pin and Toolchain lock. Updating Engine 2.6 cannot rewrite Engine 2.3 files or a project lock referencing it.

Packages shared by multiple Toolchains are reference-counted. Removing one Toolchain owner does not delete bytes still referenced by another.

## 6. Thin shim responsibilities

The permanent shim:
1. locates the intended project context;
2. reads project Engine pin or explicit Engine override;
3. locates the compatible Engine binary;
4. checks Toolchain lock and missing required SDK packages;
5. invokes the correct Engine;
6. forwards arguments, exit status and signals.

It remains small and stable. Heavy project parsing/build logic belongs to Engine, not the shim.

## 7. Locking and mutation

All Toolbelt mutations use one cross-process lock for index/package registration consistency. Read-only listing may use a consistent snapshot. Lock metadata may include PID, process, command and start time for diagnostics, but correctness depends on an OS lock primitive rather than PID liveness.

The lock protects package-index updates, CAS registration/reference counts, install/update/repair/remove/prune transactions and Toolchain lock changes involving local package state.

Build reads should not block once immutable packages are registered.

## 8. Toolchain lifecycle

- Resolve: compute required package closure and compatibility.
- Inspect: show exact Engine/SDK/component identities and reasons.
- Lock: persist immutable IDs and hashes.
- Verify: ensure every locked package exists and passes integrity checks.
- Update: report newer compatible choices without silently rewriting pins.
- Upgrade: explicitly choose a new Toolchain and record the diff.
- Remove: release the Toolchain's ownership references.
- Repair: restore missing/corrupt immutable packages.

## 9. Acceptance criteria

- [ ] Multiple Engine versions coexist.
- [ ] Multiple project Toolchains coexist.
- [ ] A shared SDK package is not removed while referenced.
- [ ] Toolbelt package state is distinct from one project's Toolchain.
- [ ] Shim routing follows explicit project pins.
- [ ] Mutations use a cross-process lock and atomic metadata update.
