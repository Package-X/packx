# PackageX Compatibility Registry

## 1. Purpose

The Compatibility Registry publishes tested relationships between PackageX Engine versions and SDK packages. PackX uses it to resolve supported combinations without guessing from version numbers or downloading every SDK for inspection.

It is a compatibility data service, not usage telemetry.

## 2. Compatibility dimensions

A supported combination must satisfy:
1. **SDK API compatibility:** Engine and SDK speak a compatible BuildRequest/BuildResult contract.
2. **Engine/package compatibility:** SDK package declares a tested Engine range or exact supported releases.
3. **Target/capability compatibility:** package supports the requested OS, architecture, format and operation set.

A numeric SDK version alone is not sufficient. A version may be in range but lack a requested writer operation or target.

## 3. Engine declaration

Each Engine release publishes:
- Engine version and build identity;
- SDK API version/range;
- supported SDK package IDs and version ranges;
- required package/capability mapping per target and format;
- optional capabilities and feature-to-component mapping;
- minimum host requirements;
- compatibility test matrix or evidence revision.

Engine remains authoritative for project interpretation. PackX asks Engine for required target/capability set and uses registry metadata to resolve package identities.

## 4. SDK declaration

Each SDK package publishes:
- package ID, version and Stable/Nightly identity;
- SDK API compatibility;
- supported/tested Engine versions/range;
- target OS/architecture/ABI;
- supported formats and operations: create, inspect, modify, transform, verify, sign;
- required/optional component dependencies;
- minimum host support;
- known limitations and experimental capabilities;
- manifest digest, full source commit and provenance.

## 5. Illustrative registry record

~~~json
{
  "schema": "packagex-compatibility",
  "schema_version": 1,
  "revision": 17,
  "engines": [
    {
      "version": "2.3.0",
      "sdk_api": "1",
      "supported_sdk": [
        {
          "package_id": "px-sdk-windows-x64",
          "range": ">=2.3.0 <2.7.0",
          "targets": ["windows-x86_64"],
          "formats": ["exe", "msi"],
          "operations": ["create", "inspect", "verify"]
        }
      ]
    }
  ],
  "packages": [],
  "revocations": []
}
~~~

Example only, not frozen wire schema. Canonical encoding, signature format, revision/freshness semantics and range grammar must be specified before implementation.

## 6. Human-readable compatibility page

A website may present:

~~~text
PackageX Engine 2.3
  SDK packages: 2.3 - 2.6
  Windows x64: MSI, EXE
  Linux x64: DEB
  macOS ARM64: PKG, DMG
~~~

The displayed ranges are generated from published/tested records. Never imply every target or format works with every patch release.

## 7. Privacy and telemetry boundary

Compatibility lookup requires no project source, app files, usernames, installed-file inventories or build history.

PackX requests only metadata needed to resolve package IDs and versions. Future usage analytics, if any, must be separate, opt-in and never required for installation or compatibility.

## 8. Resolution precedence

- Exact project Toolchain lock/package digest first.
- Explicit package pins next.
- Engine API/range plus requested target/capabilities constrain candidates.
- Stable recommended pointer is default.
- Nightly is eligible only after explicit user selection.

Latest pointers are mutable catalog metadata. Project locks persist immutable resolved identities.

## 9. Signed registry and revision policy

Registry/catalog metadata is signed. PackX verifies trusted key, canonical payload digest, monotonic revision/freshness policy, selected package manifest digest and revocation state.

A cached registry may be used offline only under explicit locked/offline policy and freshness rules. A stale registry cannot silently authorize new package identities.

## 10. Acceptance criteria

- [ ] Engine release declares API and package compatibility.
- [ ] SDK package declares targets, operations and Engine range.
- [ ] Registry represents tested combinations and known exclusions.
- [ ] Version arithmetic alone never establishes support.
- [ ] Lookup requires no project source upload or telemetry.
- [ ] Stable default and explicit Nightly policy are enforced.
- [ ] Registry signature, revision and revocation are verified.
