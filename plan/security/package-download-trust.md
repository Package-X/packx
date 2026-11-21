# Package Download and Trust Model

## 1. Trust objectives

PackX downloads executable Engine and SDK packages. It must establish authenticity and integrity before extracting, registering or executing them.

HTTPS alone is insufficient: a server/CDN/mirror may serve incorrect bytes over an encrypted connection.

Trust chain:
1. Locally trusted PackageX release key.
2. Signed catalog/compatibility metadata.
3. Signed SDK/Engine package manifest.
4. Full artifact digest and byte size.
5. Extracted-file inventory.
6. Verified immutable CAS package.

## 2. Catalog validation

Before resolving new package identities:
- verify catalog signature against configured trusted keys;
- verify canonical payload digest;
- enforce revision/freshness policy;
- reject malformed schema/unsupported major schema;
- check package revocations;
- validate package/target/channel references.

A catalog cache is not trusted merely because it was downloaded previously. Offline use follows explicit freshness and lock policy.

## 3. Artifact download

- Use HTTPS by default.
- Allow configured official origins or administrator-approved mirrors.
- Resolve exact version/commit identity from signed metadata.
- Stream to a unique temporary file with bounded buffers.
- Enforce declared maximum size and response limits.
- Retry transient failures with bounded backoff.
- Never log signed URL credentials/query parameters.
- A mirror cannot replace the digest or manifest signed by PackageX.

## 4. Hash and signature verification

For each package:
1. Validate package manifest signature and release-key identity.
2. Check declared archive byte size.
3. Calculate full SHA-256 and compare against signed manifest.
4. Verify platform code signature where applicable.
5. Extract and hash every declared file.
6. Reject missing/extra files unless explicitly allowed.
7. Confirm executable type, OS, architecture and ABI.
8. Register only after all checks pass.

A truncated digest may label a cache directory but never replaces full verification.

## 5. Safe archive extraction

Treat archive paths and metadata as untrusted. Reject:
- absolute, drive-qualified or parent-traversal paths;
- device nodes and unsupported special files;
- symbolic/hard links escaping package root;
- duplicate conflicting paths;
- case-folding collisions on case-insensitive filesystems;
- excessive file count or expanded size;
- suspicious compression ratio/archive bombs;
- unexpected executable/setuid permissions;
- symlink chains escaping package root.

Extract only into a private temporary destination. Do not follow untrusted links.

## 6. Execute/load boundary

A package is not executed or dynamically loaded before verification and atomic registration.

For worker executables, provide explicit package root/request ID, controlled resource/output capabilities, restricted environment/handles, bounded time/memory where supported, cancellation and process cleanup.

For in-process libraries, load only verified immutable package objects. The exact transport remains an Engine/SDK architectural decision.

## 7. Credentials and secrets

- No private signing keys, passwords or raw credentials in Toolbelt index, Toolchain lock, manifests or logs.
- Publisher keys use platform key store or approved external signer.
- Catalog signing key is available only in protected release workflow stage.
- Download tokens/signed URLs are short-lived and not persisted as package metadata.
- Redact credentials from URLs, environment variables and diagnostic paths.

## 8. Recovery and revocation

- Invalid temporary downloads are deleted/quarantined, never registered.
- Corrupt installed package is marked invalid and excluded from new resolution.
- Repair fetches exact immutable identity.
- Revoked identities are not newly resolved.
- Existing pins are not silently rewritten; report revocation and require explicit migration/override.
- Active SDK processes hold leases preventing garbage collection.

## 9. Audit record

Record package ID, visible identity, full Nightly SHA, manifest digest, artifact hashes, catalog revision, signing key ID, source/mirror origin, install time, Toolchain owners and verification result.

Do not record project source or secrets.

## 10. Acceptance criteria

- [ ] Invalid catalog signatures fail closed.
- [ ] Artifact digest mismatch fails before extraction/registration.
- [ ] Mirrors cannot redefine trusted digests.
- [ ] Traversal/bomb/path collision archives are rejected.
- [ ] Package is not loaded before verification.
- [ ] Catalog rollback is detected.
- [ ] Repair preserves immutable identities.
- [ ] Secrets are absent from persisted metadata/logs.
