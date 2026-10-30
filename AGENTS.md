# AGENTS.md

## What this is

Swift package (`swift-tools-version: 6.0`, iOS 15+) wrapping Apple's `Security`
framework (`SecTrust`) for X.509 chain validation plus a hand-rolled ASN.1/X.509
parser for certificate field inspection. It is consumed from Kotlin/Native via
cinterop (the EUDI wallet KMP stack, `eudi-lib-kmp-etsi-1196x2`) through
Kotlin Toolchain's native `swiftPackage:` dependency mechanism — all public
types are `@objc` classes so they bridge to Objective-C/Kotlin, not plain
Swift structs/enums. The minimum iOS version is tied to the Swift runtime's
back-deployment compatibility shims (see "Deployment target" below) — don't
lower it without checking that.

## Critical: the package does NOT build from `Sources/`

`Package.swift` declares `PKIXBridge` as a **`.binaryTarget`** pointing at
`./package/PKIXBridge.xcframework`. That directory is **not checked in** —
it only exists after `scripts/build-xcframework.sh` has run (locally, or via
the `Release` GitHub Actions workflow). `Sources/*.swift` is the real source
of truth for the binary, but it is compiled only by that script, never by
`swift build`/`swift test` directly — those commands need `package/` to
already be populated or they fail with a missing binary target.

- `Tests/PKIXBridgeTests` depends on the `PKIXBridge` binary target, so it
  exercises whatever is currently sitting in `package/PKIXBridge.xcframework`
  — run `scripts/build-xcframework.sh` first if you've changed `Sources/`.
- Editing `Sources/PKIXBridge/*.swift` has **zero effect** until you
  (re)run `scripts/build-xcframework.sh` (macOS + Xcode command line tools
  required) to regenerate `package/PKIXBridge.xcframework`, or until the
  `Release` workflow does it in CI.
- The published/committed xcframework is produced and pushed back to `main`
  only by `.github/workflows/release.yml` (`workflow_dispatch`, macOS
  runner) — it builds via the script, zips the result, computes a checksum,
  and commits `package/PKIXBridge.xcframework` with `[skip ci]`. There is no
  automatic release on every push; someone must manually dispatch it.

## Deployment target: `MIN_IOS` in `scripts/build-xcframework.sh`

The **actual** compiled minimum iOS version comes from `MIN_IOS` in
`scripts/build-xcframework.sh` (currently `16.0`), which is passed as
`-target arm64-apple-ios${MIN_IOS}` to every `swiftc` invocation. Keep this
in sync with `platforms: [.iOS(...)]` in `Package.swift` — the latter is
only a SwiftPM-level declaration for consumers' type-checking and does not
by itself change what deployment target the binary was actually compiled
for.

- The framework is built `-emit-library -static` — a static archive, not a
  dynamic framework. Static Swift binaries embed `__swift_FORCE_LOAD_$_*`
  autolink symbols for Swift ABI back-deployment compatibility shims
  (`swiftCompatibility51`, `swiftCompatibility56`, `swiftCompatibilityConcurrency`,
  `swiftCompatibilityPacks`, etc.) whenever `MIN_IOS` is old enough to need
  them. Recent Xcode versions have been dropping these shims entirely, so
  building against a very old `MIN_IOS` with a very new Xcode can produce a
  binary that references compatibility libraries the toolchain's linker can
  no longer find — this surfaces downstream as `ld: ... library
  'swiftCompatibilityNN' not found` / `Undefined symbols ...
  __swift_FORCE_LOAD_$_swiftCompatibilityNN` in the *consumer's* link step
  (e.g. Kotlin/Native's `linkIosX64TestDebug`), not in this repo's own build.
- If raising `MIN_IOS` doesn't fully eliminate these symbols, check which
  Swift feature introduced the specific missing shim (e.g.
  `swiftCompatibilityPacks` tracks parameter-pack ABI support, which landed
  later than concurrency) and raise further accordingly.
- Always rebuild with the same (or newer) Xcode version that the downstream
  consumer's CI uses — mismatched Xcode versions between where the
  xcframework was built and where it's linked is a common source of these
  errors even when `MIN_IOS` looks correct.

## Build/test (requires macOS + Xcode toolchain; this repo has none on Windows)

- `scripts/build-xcframework.sh` — builds `package/PKIXBridge.xcframework`
  from `Sources/PKIXBridge` for iOS device (arm64) + simulator (arm64 +
  x86_64, lipo'd together). Must run on macOS with Xcode command line tools
  (`xcrun`). This is the only thing that actually compiles `Sources/`.
- `swift build` / `swift test` — standard SwiftPM commands; only meaningful
  on macOS, and only after `package/PKIXBridge.xcframework` exists (see
  above). Tests link against `Security.framework`.
- No single-test shortcut beyond SwiftPM's own
  `swift test --filter <TestClassOrMethod>`.
- Test fixtures live in `Tests/PKIXBridgeTests/Fixtures/` and are wired into
  the test target via `.process("Fixtures")` in `Package.swift` — add new
  fixture files there, not as loose resources elsewhere.
- Releasing: dispatch the `Release` workflow (`workflow_dispatch` input
  `release`) manually on GitHub — it is not triggered by pushes or tags.

## Design conventions worth knowing (from code comments)

- **No `NSDictionary` with string keys across the Swift/Kotlin boundary for
  "returned" values that matter**: `PKIXValidator.validateCertificateChain`
  returns the matched trust anchor as raw `NSData` (DER), not wrapped in a
  dictionary — `NSDictionary` string keys don't reliably bridge to Kotlin
  `String` map keys via cinterop and previously caused silent data loss.
- **Never dispatch completion handlers to the main queue.** Completions in
  `PKIXValidator` fire on whatever thread they're already on. Forcing
  main-queue delivery deadlocks Kotlin coroutine consumers that block the
  main thread awaiting the result.
- Dates cross the boundary as `NSNumber`-boxed epoch seconds
  (`timeIntervalSince1970`), not `NSDate`, because Kotlin/Native's Foundation
  binding for `NSDate` doesn't expose that accessor.
- `ASN1Parser` must never trap on malicious/malformed DER (see
  `ASN1ParserTrapRegressionTests.swift`, regression suite for ticket
  `PAR_ETSI_KMP_19`) — e.g. long-form ASN.1 lengths must be bounds-checked
  before arithmetic (`offset + length`) to avoid `Int` overflow traps. Any
  change to length/offset handling in `ASN1Parser.swift` should be run against
  that test file specifically.
- `X509DistinguishedName`/subject-issuer dictionaries collapse multi-valued or
  repeated RDN attributes with "last wins" semantics — this is intentional and
  mirrored on the Kotlin side's `Map<String, String>`.
