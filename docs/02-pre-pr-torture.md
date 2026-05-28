# 02 - Pre-PR Torture

Five-level mechanical validation harness. Runs between Phase 4 (Surgical fix)
and Phase 5 (Fork + PR). A failure at any level puts the protocol in
**safe-mode**: the PR is not pushed, the failure is investigated, the level
re-runs. Only when all five pass does the run advance.

The harness exists because every level catches a class of false-positive PR
that is otherwise indistinguishable from a good PR on the local machine.

## L1 - Failing repro test BEFORE the fix is applied

The most important level and the one most often skipped.

**The contract:** write a test that demonstrates the bug, run it against the
unmodified codebase, and **observe it fail with a specific error tied to the
root cause analysis from Phase 3.** Only then apply the fix. Re-run the same
test. It must now pass.

**Why this matters:** a fix that passes a test does not prove the fix is
correct. A fix that converts a specific failing test into a passing test, where
the failure mode matches the diagnosed root cause, proves the fix addresses
the diagnosed bug. Without L1 the run is "I changed some code and the tests
that exist still pass", which is a much weaker claim.

**Output format:** a paragraph documenting the pre-fix run output (the exact
error message or assertion failure), then the post-fix run output (clean
pass), then the diff between them.

**Failure modes:**

- The test passes against the unmodified code. The test does not exercise the
  bug; the diagnosis is wrong or the test is wrong.
- The test fails against the unmodified code AND fails against the patched
  code. The fix does not solve the diagnosed root cause.
- The test fails with a different error than the one diagnosed. The diagnosis
  is incomplete; return to Phase 3.

## L2 - Full local test suite

After L1 turns green, run the full local test suite.

**Goal:** confirm the fix does not break unrelated tests.

**Mechanical:** whatever the project uses. `npm test`, `pnpm test`,
`cargo test --all`, `pytest`, `go test ./...`. Read the project README or
CONTRIBUTING for the canonical command.

**Output:** a summary line. "N tests, M passed, 0 failed" where N matches the
unmodified baseline.

**Failure modes:**

- An unrelated test fails. The fix has a wider blast radius than intended.
  Either narrow the fix, or add a justification for the broader change.
- A test that was failing before the fix is now passing. Document this as a
  positive side effect in the PR body.

## L3 - Cross-platform check

The level that catches "works on my Linux box, breaks on macOS" surprises.

**Goal:** confirm the fix is platform-agnostic, or that the platform
specificity is intentional and documented.

**Mechanical:** the local machine cannot run the full matrix. Compensate by:

- Reading the diff with explicit attention to platform-conditional code paths
  (`process.platform`, `cfg!(...)`, `#[cfg(...)]`, `if os.platform()...`).
- Searching the project for similar code patterns and verifying the fix
  follows the project's platform-handling convention.
- Confirming the fix does not rely on local-only side effects (a path
  separator, an environment variable, a locale).
- If the fix touches a platform-specific area, explicitly state in the PR
  body which platforms were not locally tested and why CI is expected to
  cover them.

**Output:** a one-sentence cross-platform claim ("retention-only change, no
platform-conditional code added, works the same on macOS and Windows where
the underlying binding already retained the wrapper").

**Failure modes:**

- The fix introduces a platform-conditional path that was not present before.
  This is a wider change than a bug fix; split it.
- The fix relies on a local side effect (path separator, environment
  variable). Rewrite to not rely on it.

## L4 - Lint and format compliance

Every project has its own style enforcement. CI will catch violations
mechanically. L4 catches them locally so the PR opens green.

**Mechanical:** run whatever the project enforces. Common tooling:

- ESLint or Biome for JavaScript / TypeScript
- Prettier or oxfmt for formatting
- rustfmt and clippy for Rust
- Black, Ruff for Python
- gofmt and golangci-lint for Go

If the project has a `.changeset/` directory, the project uses Changesets and
the PR needs a changeset file. If the project has a `CHANGELOG.md` and
requires manual entries, add one.

**Output:** "lint clean, format clean, changeset added (if required)".

**Failure modes:**

- Style tool is not installed locally. Install it. Do not skip the level.
- Style violation is in pre-existing code adjacent to the fix. Do not
  reformat the surrounding code; leave it as-is. Reformatting unrelated lines
  pollutes the diff and triggers reviewer pushback.

## L5 - PR_SUMMARY self-review

The level that catches the "this PR body is vague" failure mode.

**Goal:** the PR body should be readable by a maintainer in two minutes and
should answer: what bug, what root cause, what fix, what tests, what
trade-offs.

**Mechanical:** write the PR body, then read it as if you were a maintainer
seeing the project for the first time. Check:

- Is the root cause stated explicitly, or is the PR body only describing what
  the fix changes?
- Are alternative fixes mentioned and dismissed with reasons?
- Are the trade-offs of the chosen approach acknowledged?
- Are the regression tests described, not just listed?
- Is there any unsupported claim (e.g., "the maintainer confirmed this fix"
  when the maintainer only confirmed the bug)?

The last point is non-negotiable. Inflating reviewer language is the fastest
way to lose credibility.

**Output:** the final PR body, em-dash-free, no LLM-tell phrases, no future
tense ("the fix will resolve..."), only past tense for completed work and
present tense for facts.

**Failure modes:**

- The PR body is shorter than the diff suggests. Add the missing context.
- The PR body uses marketing register ("revolutionary fix", "elegant
  solution"). Rewrite in plain register.
- The PR body claims something the diff does not show. Rewrite or back the
  claim with evidence.

## What "safe-mode" means

If any level fails, the run does not advance to Phase 5. The cycle is:

1. Document the failure in the run log.
2. Diagnose the cause.
3. Return to whatever upstream phase produced the bad input (often Phase 3 if
   L1 fails, Phase 4 if L2 fails, Phase 4 if L3 or L4 fails, Phase 5 if L5
   fails).
4. Re-run from there forward.
5. Re-run Pre-PR Torture from L1.

Safe-mode is the protocol's defense against the worst failure mode in
upstream contribution: pushing a PR that has subtle bugs the local checks
missed, which damages trust with the maintainer long-term.
