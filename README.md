# cold-forge

A protocol for proposing surgical fixes to upstream open-source projects without
scooping community contributors, with explicit phase gates, mechanical validation,
and a file-based memory layer.

Cold Forge is a **methodology**, not a tool. The reference implementation in this
repo is one possible orchestration. The protocol itself is the artifact worth
adopting.

## What this solves

Drive-by contributions to popular open-source projects are a coordination problem,
not a coding problem. The bug is usually findable; the path between "I found the
bug" and "the maintainer merged my fix" is where contributions die. Specifically:

1. A community member already proposed a fix in the issue thread and is waiting
   for maintainer green-light. Opening your own PR scoops them and burns trust.
2. The fix passes locally but fails on a platform you do not test against, or
   fails a stylistic gate the project enforces but does not document.
3. The PR body is a generic patch description rather than a maintainer-ready
   document explaining the root cause, the alternative paths considered, and the
   trade-offs of the chosen path.
4. The repository contains undocumented build prerequisites (Rust toolchain,
   wasm-pack, a custom test runner) that cost hours to discover. Without notes,
   the next contribution session repeats the discovery.

Cold Forge addresses these as **explicit phase gates** with documented entry and
exit criteria. A failure at any gate puts the run in safe-mode rather than
silently pushing a broken PR.

## Architecture

Eight ordered phases. A run advances through them in sequence. Each phase has an
explicit failure mode and a documented escape hatch.

```
Phase 0   - Target selection      (which repo, which issue, why)
Phase 1   - Dedupe check          (existing PRs on this issue?)
Phase 2   - Issue triage          (read the full thread, not just the body)
Phase 3   - Static analysis       (find the root cause in source)
Phase 4   - Surgical fix          (minimum-blast-radius change)
Phase 4.5 - CLA preflight         (verify external contributor admissibility)
Phase 4c  - Ethical gate          (any community member in flight? defer if so)
Pre-PR Torture L1 to L5           (failing repro test, full suite, cross-platform,
                                   lint/format, PR_SUMMARY self-review)
Phase 5   - Fork + branch + PR    (Conventional Commits, sanitized identity)
Phase 6   - Maintainer signal     (response window, escalation cadence)
Phase 6b  - Continuous CI         (poll CI 180s, auto-fix mechanical failures
                                   until green, do not make maintainers chase)
Phase 7   - Persistence layer     (RAG saves on every signal change, cross-
                                   session memory, dev-env quirks as .md)
```

See [docs/01-protocol.md](docs/01-protocol.md) for each phase in detail.

## Invariants

The protocol exists to enforce three properties that are easy to lose under time
pressure:

1. **No scooping.** [Phase 4c](docs/03-ethical-gate.md) reads every comment on
   the target issue before opening a PR. Community members who proposed a fix
   and asked for maintainer green-light are not yet "claimed" in any visible
   sense, but they hold the claim socially. Cold Forge defers in their favor
   and offers Co-authored-by attribution.

2. **No false-positive PRs.** [Pre-PR Torture](docs/02-pre-pr-torture.md) is a
   five-level mechanical validation harness. L1 mandates a failing repro test
   before the fix is applied. L2 runs the full local suite. L3 cross-checks the
   fix on platforms the local environment cannot exercise. L4 enforces project
   style (eslint, prettier, oxfmt, biome, whatever the project uses). L5 is a
   self-review of the PR body for vagueness and missing root-cause analysis.

3. **No knowledge loss between sessions.** [The memory layer](docs/05-rag-memory.md)
   uses two tiers. Runtime state (PR scoreboards, CI iteration chains,
   maintainer signals, outreach handoffs) goes to an embedded RAG with a
   `memory_save(note, tag)` API, semantically searchable across sessions
   and projects. Meta-rules and per-project dev-env quirks stay as plain
   markdown files that the agent auto-loads on every session start. A
   second contribution to the same project does not repeat the discovery
   work of the first.

4. **Maintainers review content, not your CI.** [Phase 6b](docs/01-protocol.md)
   activates automatically after every push: poll CI every 180 seconds,
   triage each failure, surgical-fix mechanical errors (compile, test,
   lint, naming, format), push and re-run. The maintainer's first look at
   the PR should find all jobs green, not a chain of red Xs they have to
   help you debug. If CI cannot converge after five iterations, stop and
   ask one focused question instead of pushing more variations.

## Examples

Two worked examples, both public PRs against active upstream projects:

- [examples/anytype-2233.md](examples/anytype-2233.md): Date Format picker
  ambiguity in anyproto/anytype-ts. Five-line fix, two regression tests, merged
  in four hours by the file maintainer.

- [examples/tutanota-10881.md](examples/tutanota-10881.md): Linux click-to-update
  notification GC race in tutao/tutanota. Twenty-seven-line fix, two regression
  tests, retention pattern mirroring an existing function in the same file.

## What this is not

- Not a static analyzer. The "find" step is exploratory and uses standard
  tooling (grep, git blame, file reading).
- Not a code generator. Every diff is human-reviewed before submission.
- Not an outreach automation. The protocol covers PR submission only. Cold-mail
  workflow is out of scope and not included in this repo.
- Not project-specific. The eight phases are repo-agnostic. Per-project
  knowledge lives in the memory layer, not in the protocol.

## License

MIT.
