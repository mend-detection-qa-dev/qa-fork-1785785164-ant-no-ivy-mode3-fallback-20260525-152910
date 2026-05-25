# ant-no-ivy-mode3-fallback

Purpose-built **Mode-3 native** probe for the Mend SCA detection QA suite.
This is NOT a Mode-1 probe that pivoted — there is no Ivy at all.

## Catalog patterns covered

| ID   | Pattern name                              | What it exercises |
|------|-------------------------------------------|-------------------|
| E3   | `ant-build-xml-without-ivy`               | `build.xml` with `<path id="lib.cp">` + `<fileset dir="lib">` and no `<ivy:*>` task |
| G6.b | `ant-default-path-scan-mixed-extensions`  | `lib/` contains `.jar`, `.war`, `.zip`, `.so` — the full default Mode-3 extension set |
| G6.c | `ant-path-id-includes-wildcard`           | `whitesource.config` sets `ant.pathIdIncludes=lib.*` to filter on Ant Path ID wildcard |

## Feature exercised

Mend's Ant Mode-3 default path scan: the scanner fingerprints binary files directly under
paths declared by `ant.pathIdIncludes`, without invoking Ivy or resolving any manifests.
Every file in `lib/` is committed and detected via SHA1 fingerprint.

## Project structure

```
build.xml              — single-module Ant build; declares <path id="lib.cp">
.whitesource           — Bucket A config; pins java=17.0.19+10
whitesource.config     — ant.resolveDependencies=true, ant.pathIdIncludes=lib.*
.gitignore             — lib/ is NOT excluded (binaries committed intentionally)
expected-tree.json     — expected flat dependency tree (8 direct deps)
lib/
  commons-io-2.11.0.jar        — real Maven Central JAR
  commons-lang3-3.12.0.jar     — real Maven Central JAR
  commons-logging-1.2.jar      — real Maven Central JAR
  junit-4.13.2.jar             — real Maven Central JAR
  slf4j-api-1.7.36.jar         — real Maven Central JAR
  test-archive.war             — renamed copy of commons-io-2.11.0.jar (.war extension)
  test-archive.zip             — renamed copy of commons-logging-1.2.jar (.zip extension)
  test-binary.so               — renamed copy of slf4j-api-1.7.36.jar (.so extension)
```

## lib/ file inventory

| File | Extension | Bytes | SHA1 | Source |
|------|-----------|-------|------|--------|
| `commons-io-2.11.0.jar` | `.jar` | 327135 | `a2503f302b11ebde7ebc3df41daebe0e4eea3689` | Maven Central (real) |
| `commons-lang3-3.12.0.jar` | `.jar` | 587402 | `c6842c86792ff03b9f1d1fe2aab8dc23aa6c6f0e` | Maven Central (real) |
| `commons-logging-1.2.jar` | `.jar` | 61829 | `4bfc12adfe4842bf07b657f0369c4cb522955686` | Maven Central (real) |
| `junit-4.13.2.jar` | `.jar` | 384581 | `8ac9e16d933b6fb43bc7f576336b8f4d7eb5ba12` | Maven Central (real) |
| `slf4j-api-1.7.36.jar` | `.jar` | 41125 | `6c62681a2f655b49963a5983b8b0950a6120ae14` | Maven Central (real) |
| `test-archive.war` | `.war` | 327135 | `a2503f302b11ebde7ebc3df41daebe0e4eea3689` | JAR bytes renamed (same SHA1 as commons-io-2.11.0.jar) |
| `test-archive.zip` | `.zip` | 61829 | `4bfc12adfe4842bf07b657f0369c4cb522955686` | JAR bytes renamed (same SHA1 as commons-logging-1.2.jar) |
| `test-binary.so` | `.so` | 41125 | `6c62681a2f655b49963a5983b8b0950a6120ae14` | JAR bytes renamed (same SHA1 as slf4j-api-1.7.36.jar) |

### Non-JAR extension strategy

Real `.so`, `.dll`, `.exe` files with verified Mend KB SHA1s are not publicly available.
The probe uses a documented rename strategy: real Maven Central JAR bytes are copied and
renamed to the target extension. SHA1 is byte-invariant under rename, so:

- Mend fingerprints the file, computes the correct SHA1.
- KB lookup may match the original artifact (e.g., `test-archive.war` may be reported as
  `commons-io-2.11.0`). This is acceptable: the detection path (file seen, SHA1 computed)
  is exercised regardless of KB match.
- The primary assertion for non-JAR files is presence in the dependency tree with the
  correct SHA1. Name matching is secondary and ecosystem-dependent.

## Expected dependency tree

All 8 files in `lib/` should appear as flat direct dependencies with no transitive structure.
Source is `local` for all entries. Versions are extracted from filenames for real JARs; for
renamed non-JAR files the version is that of the underlying JAR content.

See `expected-tree.json` for the full tree with SHA1 hashes.

## Why ant.ivyResolveDependencies is not set

`ant.ivyResolveDependencies` defaults to `false` and is intentionally absent from
`whitesource.config`. This ensures Mend's Ant resolver does NOT attempt Mode-1 Ivy
resolution. The probe validates Mode-3 purely. Setting the flag to `true` without a valid
`ivy.xml` would cause the resolver to error and fall through — an undefined state. The
clean probe keeps the modes separate.

## Mend config

**Bucket A — default-emit.** `.whitesource` pins `java` to `17.0.19+10` via
`scanSettings.versioning`. Ant itself is NOT pinnable via `install-tool` — there is no
`ant` key in Mend's `scanSettings.versioning` registry. The Ant binary version is
operator-supplied out-of-band. This is a **partial reproducibility limitation**: scans run
against different Ant versions may produce different transitive sets IF Ant version affects
file discovery (it does not in Mode-3, but the limitation is noted for completeness.

No additional `.whitesource` dimensions are required:
- No branch/project-token scoping needed (default-branch scan).
- No `whitesource.config` external URL (file is at repo root, default discovery applies).
- `configMode: "LOCAL"` in `.whitesource` — UA reads `whitesource.config` from repo root.

## Probe metadata

- Generated: 2026-05-25
- Probe ID: ant-no-ivy-mode3-fallback-20260525-152910
- PM: ant (Mode-3, no Ivy)
- Schema version: 1.0
- Catalog IDs: E3, G6.b, G6.c
