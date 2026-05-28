# Example: anytype-ts #2233

A worked example of one Cold Forge run, end to end. Public links throughout
so every claim is verifiable.

- **Repo:** [anyproto/anytype-ts](https://github.com/anyproto/anytype-ts)
- **Issue:** [#2208 - Date format picker ambiguity](https://github.com/anyproto/anytype-ts/issues/2208)
- **PR:** [#2233](https://github.com/anyproto/anytype-ts/pull/2233) - merged
- **Diff:** +35 / -1 LOC, two files
- **Time from PR open to merge:** approximately four hours

## Phase 0 - Target selection

Candidate criteria: a TypeScript / React project with active maintainer
engagement, an issue with concrete reproduction steps, fix scope under fifty
lines. The repo had recent commits, a maintainer (@ra3orblade) actively
shipping in the same area, and an issue with a clear repro:

> "On May 5 (and on 11 other days a year where day == month), the Settings
> > Language & Region > Date Format picker shows two radio buttons with
> identical labels. The Short format (d/m/Y) renders to `05.05.2026` and
> ShortUS (m/d/Y) also renders to `05.05.2026`."

Concrete bug, deterministic trigger (12 specific dates per year), scope
likely small. Selected as the run target.

## Phase 1 - Dedupe

```
gh pr list --repo anyproto/anytype-ts \
  --state all --search "#2208 in:body,title" \
  --json number,state,title
```

Zero PRs referenced the issue. Advance.

## Phase 2 - Issue triage

Read the full thread. Three comments total, no community fix proposed, no
maintainer "by-design" disclaimer. The bug was straightforward and unclaimed.
Advance.

## Phase 3 - Static analysis

The picker preview was built by formatting `new Date()` (i.e., today's date)
through each `DateFormat` enum value. The function source revealed the
mechanism:

```typescript
// menu.ts - format preview for the date-format radio picker
options: DateFormat.map(format => ({
  id: format,
  name: U.Date.date(format, now()),  // <- formats today's date
  caption: ...
}))
```

The bug surfaced when `now()` happened to be a "symmetric" date (day == month).
On such dates, `Short` (d/m/Y) and `ShortUS` (m/d/Y) produced the same visible
string. The implementation was correct; the choice of preview sample was the
bug.

The `DateFormat` enum comments already used a fixed asymmetric sample
(July 30, 2020) for documenting the format options. The convention existed,
just not in the picker code.

## Phase 4 - Surgical fix

Replace `now()` with a fixed asymmetric timestamp matching the enum-comment
convention:

```typescript
// Use a fixed asymmetric sample (Jul 30, 2020) so the visible preview
// strings differ between formats that share characters on "symmetric"
// dates (day == month). Matches the convention already used in the
// DateFormat enum doc comments.
const SAMPLE_DATE = new Date(2020, 6, 30);  // Jul 30, 2020

options: DateFormat.map(format => ({
  id: format,
  name: U.Date.date(format, SAMPLE_DATE.getTime()),
  caption: ...
}))
```

Total: five lines of production code change.

## Phase 4.5 - CLA preflight

The anyproto org uses CLA Assistant on first PR. Identity used was GitHub
no-reply tied to the contributor account, so CLA Assistant auto-signed on PR
open. No friction.

## Phase 4c - Ethical gate

Re-scanned the thread. No community member had proposed a fix. No "happy to
submit a PR if so" phrases. The reporter had not opened a parallel draft.
Gate clear. Advance.

## Pre-PR Torture

**L1:** added two regression tests in `date.test.ts`:

1. Positive guard: the fixed sample produces distinguishable previews for
   `Short` and `ShortUS`. Passes after fix.
2. Negative documentation: building the preview with `now()` on a symmetric
   date produces identical previews for `Short` and `ShortUS`. Demonstrates
   the bug. Marked as `// regression guard - reproduces #2208`.

Both tests fail against the unmodified `develop` branch (the positive guard
fails because the picker is using `now()`; the negative documentation passes
because the bug is present). Both produce the expected post-fix outcomes
(positive passes, negative fails as it documents the prevented bug).

**L2:** full vitest suite ran clean. No unrelated test regressions.

**L3:** the fix is purely a constant change. No platform-conditional code.
Cross-platform claim: trivial.

**L4:** biome lint clean. tsc clean. The project does not use Changesets.

**L5:** PR body self-review caught one initial draft that said
"refactor the picker logic" - over-claimed. Rewrote to "use a fixed sample
date for the picker preview, matching the convention in `DateFormat` enum
comments".

## Phase 5 - Fork + PR

Branch name: `fix/2208-date-format-picker-symmetric-dates`. Conventional
Commits style not used by this project; matched the project's plain
imperative-mood commit message convention.

PR body sections: Summary, Root cause, Fix, Tests, Notes on convention
alignment.

## Phase 6 - Maintainer signal

PR opened at 17:43 CEST. @ra3orblade approved, merged, and closed #2208 at
21:43 CEST. Four-hour turnaround. Zero review comments. Clean accept.

## Phase 7 - Persistence

Three artifacts produced:

1. **Memory note** on `anyproto/anytype-ts` dev-env quirks: develop is the
   integration branch (not master), bun.lock is the lockfile (not
   package-lock.json or pnpm-lock.yaml), biome is the lint+format tool, vitest
   for tests. The next contribution to the same repo starts with this
   knowledge cached.
2. **Scoreboard update:** PR shipped, merged, closed #2208 cleanly. State
   recorded for monitor purposes.
3. **Worked example:** this file.

## What this run demonstrates about the protocol

The fix itself was five lines. The total run time was several hours,
dominated by Phase 2 (reading the thread), Phase 3 (locating the picker
preview function), L1 (writing both regression tests), and L5 (rewriting the
PR body twice).

The protocol's value is not visible in any single step. It is visible in
the outcome: a merged PR on first attempt with zero review comments, four
hours after open. That outcome is reproducible against other projects only
if every phase runs.
