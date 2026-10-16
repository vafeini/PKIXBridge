# AGENTS.md

## What this is

Swift package (`swift-tools-version: 6.0`, iOS 13+) wrapping Apple's `Security`
framework (`SecTrust`) for X.509 chain validation plus a hand-rolled ASN.1/X.509
parser for certificate field inspection. It is consumed from Kotlin/Native via
cinterop (likely the EUDI wallet KMP stack) — all public types are `@objc`
classes so they bridge to Objective-C/Kotlin, not plain Swift structs/enums.

## Critical: the package does NOT build from `Sources/`

`Package.swift` declares `PKIXBridge` as a **`.binaryTarget`** pointing at the
prebuilt, committed archive `distribution/archive/PKIXBridge.xcframework.zip`
(expanded copy also checked in under `distribution/PKIXBridge.xcframework/`).
Only the `PKIXBridgeTests` test target actually compiles `Sources/`... wait,
no — check again: the test target depends on `PKIXBridge` (the binary target),
so **`Sources/*.swift` is not compiled or exercised by `swift test` at all**.
It currently exists only as reference/history for what the binary was built
from.

- If you edit files under `Sources/PKIXBridge/`, those changes have **no
  effect** on `swift build` / `swift test` / the exported xcframework until
  someone rebuilds the `.xcframework` (e.g. via `xcodebuild -create-xcframework`
  or `swift build` targeting the actual source + packaging into the archive)
  and replaces both `distribution/PKIXBridge.xcframework/` and
  `distribution/archive/PKIXBridge.xcframework.zip`. There is no script in
  this repo that does this — it must be done manually/externally.
- Before investing effort editing `Sources/`, confirm with the user whether
  they want source changes (which require a manual rebuild+repackage step
  outside this repo) or whether the task is actually about the binary/tests.
- Do not assume `swift test` validates your `Sources/` edits — verify by
  checking which target actually got compiled.

## Build/test (requires macOS + Xcode toolchain; this repo has none on Windows)

- `swift build` / `swift test` — standard SwiftPM commands, but only
  meaningful on macOS with a matching Xcode/Swift toolchain since the target
  is an iOS binary framework and tests link against `Security.framework`.
  There is no CI config, Makefile, or script in this repo — these are the
  only entry points.
- No single-test shortcut is configured beyond SwiftPM's own
  `swift test --filter <TestClassOrMethod>`.
- Test fixtures live in `Tests/PKIXBridgeTests/Fixtures/` and are wired into
  the test target via `.process("Fixtures")` in `Package.swift` — add new
  fixture files there, not as loose resources elsewhere.

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
