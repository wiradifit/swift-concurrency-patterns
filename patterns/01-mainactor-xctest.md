# Pattern 01 — @MainActor boundaries for XCTest suites

**Bug class:** Swift 6 strict-concurrency compile failure · **Found in the wild:** XPadInput (see PR #70)

## Problem

A type isolated to the main actor:

```swift
@MainActor public final class ProgressTracker {
    public func getLessonMastery(for lessonId: UUID) -> LessonMastery? { ... }
}
```

…is called from a plain (nonisolated) XCTestCase method:

```swift
func testLessonMastery() {
    let tracker = ProgressTracker.shared
    if let updated = tracker.getLessonMastery(for: id) { ... }  // ❌ error
}
```

Under Swift 6 this is a hard error, not a warning:

```
error: call to main actor-isolated instance method 'getLessonMastery(for:)'
in a synchronous nonisolated context
```

## Wrong fixes (avoid)

- `nonisolated` on the source method — silently moves UI state off the main actor; races at runtime.
- Disabling strict concurrency (`SWIFT_STRICT_CONCURRENCY=minimal`) — hides the bug, ships it to users.
- Wrapping the call in `MainActor.assumeIsolated` in tests that are *not* on the main actor — crash trap.

## Correct fix

Annotate the test class. XCTestCase supports main-actor test methods natively:

```swift
@MainActor
final class XPadPracticeTests: XCTestCase {
    func testLessonMastery() { ... }   // ✅ runs on the main actor
}
```

For async suites, prefer `@MainActor func testX() async throws` per-method so only the tests touching UI state pay the isolation cost.

## Why it matters

The compiler is telling you the truth: your test mutates shared main-actor state from wherever XCTest's default executor happens to run it. Annotating makes the executor contract explicit and keeps production isolation intact.

*Verified: annotating the class cleared all suite errors on macOS 15 / Xcode latest-stable.*
