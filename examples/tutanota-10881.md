# Example: tutanota #10881

A worked example of one Cold Forge run on a bug whose root cause was a
JavaScript engine garbage-collection race against a platform-specific binding
behavior. Picks up a class of bug that simpler protocols would miss in
Phase 3.

- **Repo:** [tutao/tutanota](https://github.com/tutao/tutanota)
- **Issue:** [#10844 - Linux: Click-to-update notification not functional](https://github.com/tutao/tutanota/issues/10844)
- **PR:** [#10881](https://github.com/tutao/tutanota/pull/10881)
- **Diff:** +71 / -2 LOC, two files
- **Status at time of writing:** open, awaiting maintainer review

## Phase 0 - Target selection

The repo had two attractive signals: a core maintainer (@charlag) commenting
directly on the issue admitting the bug was personally annoying ("For me it
is inconsistent, sometimes it works, sometimes not, slightly infuriating"),
and a bounded scope (Electron notification handler, Linux-specific). A
maintainer who calls a bug "infuriating" carries strong merge incentive once
a clean fix lands.

Selected as the run target.

## Phase 1 - Dedupe

```
gh pr list --repo tutao/tutanota --state all \
  --search "#10844 OR \"click-to-update\" OR \"update notification\"" \
  --limit 10
```

Two historical PRs from years ago in the same area, both merged. No open PR
referenced #10844. Advance.

## Phase 2 - Issue triage

Four comments on the issue:

1. **ganthern (maintainer):** suggested workaround via Settings > Desktop >
   Update button. The notification itself was not the only path.
2. **kborch (reporter):** confirmed the workaround worked. "So it's only via
   clicking on the notification that doesn't."
3. **@charlag (core maintainer):** "For me it is inconsistent, sometimes it
   works, sometimes not. Slightly infuriating."
4. **kborch:** speculated about a version regression.

Two desktop-area maintainers acknowledged the bug. No community member
proposed a fix. No "happy to submit a PR" claim. Ethical gate clear.

The thread had a critical insight: @charlag said "inconsistent". That word
ruled out a deterministic logic bug and pointed at a timing or memory bug.
A bug that "sometimes works" is almost never a missing-if-statement; it is
usually a race or a GC-eligible reference.

## Phase 3 - Static analysis

Traced the notification flow:

1. `ElectronUpdater.notifyAndInstall` calls
   `notifier.showOneShot({title, body, icon, onClick: () => this.installUpdate()})`
2. `DesktopNotifier.showOneShot` calls `factory.makeNotification(params, onClick)`
3. `ElectronNotificationFactory.makeNotification`:
   ```typescript
   const notification = new Notification({title, icon, body}).on("click", () => onClick())
   notification.show()
   return () => notification.removeAllListeners().close()  // <- Dismisser
   ```

The factory returns a `Dismisser` closure that retains a reference to the
underlying `electron.Notification`.

Then in `DesktopNotifier.showOneShot`:

```typescript
factory.makeNotification(params, onClick)  // <- return value discarded
```

The dismisser was discarded. As soon as `showOneShot` returned, the only
reference to the `electron.Notification` went out of scope.

Cross-checked with the working path: in the same file, `showCountedUserNotification`
retains its dismisser in `notificationDismissersPerUser` map. Badge-counted
notification clicks worked correctly. One-shot notification clicks were
broken on Linux. The asymmetry between the two functions in the same file
made the root cause concrete.

Why Linux only: macOS Cocoa NSUserNotification and Windows Toast retain the
JS wrapper through their native bindings. GTK does not strongly retain the
JS wrapper across the asynchronous click round-trip. When V8 happened to
collect the wrapper before the click event landed, the click was silently
dropped. The "inconsistent" pattern @charlag described was V8 GC timing.

## Phase 4 - Surgical fix

Two changes mirroring the retention pattern already in
`showCountedUserNotification`:

1. New `private readonly oneShotDismissers: Set<Dismisser> = new Set()`
   member on `DesktopNotifier`.
2. Wrap the user-provided `onClick` so it runs the user callback, removes
   itself from the set, and calls the dismisser to close the OS
   notification:

```typescript
let dismisser: Dismisser | undefined
const onClick = () => {
    try {
        userOnClick()
    } finally {
        if (dismisser !== undefined) {
            this.oneShotDismissers.delete(dismisser)
            const toDismiss = dismisser
            dismisser = undefined
            toDismiss()
        }
    }
}
dismisser = factory.makeNotification(params, onClick)
this.oneShotDismissers.add(dismisser)
```

Total: 27 lines of production code change.

## Phase 4.5 - CLA preflight

Tutao does not use a CLA bot. Verified against recent merged PRs from
external contributors (qureshi96, hrb-hub). No friction.

## Phase 4c - Ethical gate

Re-scanned the thread. Two maintainers acknowledged the bug. Neither
proposed a fix. No community contributor in flight. Gate clear.

## Pre-PR Torture

**L1:** two regression tests added to `DesktopNotifierTest.ts`:

```typescript
o.test("showOneShot retains the notification and closes it when the user clicks (issue #10844)", async function () {
    // ... setup
    created.click()
    o.check(onClickInvoked).equals(true)
    verify(created.close(), { times: 1 })  // <- the bug check
})
```

The mock factory's `makeNotification` returns `() => n.close()` as the
dismisser. Pre-fix, `showOneShot` discarded the dismisser, so `close()`
was never called. Post-fix, the wrapped click handler invokes the dismisser,
so `close()` is called exactly once.

The test fails pre-fix with "Unsatisfied verification on test double:
`close()` wanted 1 time, no invocations". Same error fails the concurrent-
notifications variant. Post-fix: both pass.

**L2:** the Tutanota test runner has a heavy build prerequisite (Rust + wasm-pack
for the crypto-primitives crate). Local Windows machine could not run the
full pipeline. Compensated by writing a standalone tsx runner that imports
`DesktopNotifier` and `testdouble` directly, bypassing the wasm build. Ran
the four pre-existing `DesktopNotifierTest` cases and the two new ones:
7 passed, 0 failed. Limitation acknowledged in the PR body: full pipeline
validation comes from upstream CI.

**L3:** the fix is retention-only. No platform-conditional code added. The
retention has no effect on macOS or Windows (the OS bindings already retained
the wrapper). On Linux the new retention provides what GTK was not.

**L4:** ESLint clean, Prettier clean on both touched files.

**L5:** PR body self-review caught one initial draft that said "the core
maintainer confirmed the technical impact of my fix". That was inflation:
@charlag confirmed the bug existed, not the fix. Rewrote to "@charlag
confirmed the bug as 'slightly infuriating'; fix mirrors the existing
retention pattern from showCountedUserNotification; PR open for review."

## Phase 5 - Fork + PR

Branch: `fix-oneshot-notification-click-linux`.

Conventional Commits style is not strictly enforced by Tutao but recent
merges use `[area] description, close #N`. Matched:

> `[desktop] fix click-to-update notification on Linux, close #10844`

PR body sections: Summary, Root cause, Fix, Tests, Validation, Notes on the
bounded retention leak.

## Phase 6 - Maintainer signal

PR open. Awaiting review.

## Phase 7 - Persistence

Memory artifacts produced:

1. **Dev-env quirks:** tutao/tutanota is npm-based (not pnpm), branch model
   is master (not develop or main), custom @tutao/otest test framework
   bundled by the test runner, full local build requires Rust + wasm-pack
   for the crypto-primitives crate. Windows-specific gotcha: the test
   runner's `node make ${path}` call mangles Windows backslash paths
   through zx's PowerShell escape; workaround is to forward-slash the path
   locally (do not push that workaround upstream).
2. **Scoreboard update:** PR open, awaiting charlag review.
3. **This worked example.**

## What this run demonstrates about the protocol

Three classes of mistake the protocol prevented:

1. **Phase 3 could have stopped at "the click handler is wired correctly,
   the bug must be platform-specific."** The protocol mandates reading the
   sibling function (`showCountedUserNotification`) and noticing the
   asymmetry. That noticing is what produced the root cause.
2. **L1 could have been skipped** because "the bug is platform-timing-
   dependent, no test can deterministically reproduce it." The protocol's
   answer: the timing bug is a symptom of a structural bug (discarded
   reference). The structural bug IS deterministically testable by checking
   whether the dismisser is invoked on click. Write the structural test.
3. **L5 could have shipped** the original PR body with "the maintainer
   confirmed the technical impact of my fix". Any CTO or future maintainer
   reading the PR could click and see no review approval, immediately
   recognizing the inflation. The protocol's self-review caught it before
   push.

The fix itself is small. The discipline that produced the fix correctly is
the artifact.
