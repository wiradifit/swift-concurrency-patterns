# Pattern 03 — Break retain cycles in closures and timers

**Bug class:** memory leak · **The silent killer:** no crash, no log — just growing memory and degraded UX

## Problem

```swift
class PriceFeed {
    var timer: Timer?
    func start() {
        timer = Timer.scheduledTimer(for: 1.0, repeats: true) { _ in
            self.refresh()      // ❌ timer strongly holds self; self holds timer → cycle
        }
    }
}
```

`self` keeps the timer alive; the timer's closure keeps `self` alive. Neither
ever deallocates — the screen leaks on every navigation into it.

## Correct fixes

```swift
// 1. weak self + guard (closures)
timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
    self?.refresh()
}

// 2. Prefer structured concurrency — no closure, no cycle
func start() async throws {
    while !Task.isCancelled {
        try await Task.sleep(for: .seconds(1))
        await refresh()
    }
}

// 3. For Combine: .store(in:) bags + [weak self] on every sink
```

## Audit rule of thumb

Grep every escaping closure capturing `self` implicitly (Swift makes you
write `self.` explicitly — treat that keyword as a review trigger). Ask:
"who owns whom?" If the answer loops, break it with `[weak self]`, restructure
with async/await, or use an ownership-free token (`[weak self]` +
invalidation on deinit for delegates/datasources).

*Field note: sweeps across stats, Rectangle, Charts, iina found all Timer/Ticker closures correctly using `[weak self]` — the pattern is well-known but regresses constantly under deadline pressure.*
