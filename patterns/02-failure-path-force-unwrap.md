# Pattern 02 — Never force-unwrap on failure paths

**Bug class:** runtime crash · **Frequency:** the most common crash-class finding in OSS Swift audits

## Problem

```swift
func applyDiscount(code: String?) -> Int {
    let valid = validate(code!)          // ❌ crashes when code == nil
    return valid ? price / 2 : price
}
```

Force-unwraps (`!`), force-tries (`try!`), and forced casts (`as!`) convert
"unexpected input" into "process death". In shipping apps they become the
top crash-report entries; in libraries, a single hostile input becomes a DoS.

## Correct fixes

```swift
// 1. Guard — fail loudly but gracefully
guard let code, let valid = validate(code) else {
    return price
}

// 2. Domain-specific error
enum DiscountError: Error { case invalidCode }
func applyDiscountStrict(code: String?) throws -> Int {
    guard let code, validate(code) else { throw DiscountError.invalidCode }
    return price / 2
}

// 3. map/compactMap pipelines for optionals
let normalized = code.map(validate)
```

## Audit rule of thumb

`!` is acceptable only where failure is *impossible by construction*:
interface-builder outlets after `viewDidLoad`, `Bundle.main` in an app target,
literals. Everywhere else it is a latent crash ticket.

*Source: recurring class in bug-bounty sweeps (e.g. Rectangle's TimeoutCache was analyzed and cleared — the unwrap there is construction-safe; most repos are not so lucky).*
