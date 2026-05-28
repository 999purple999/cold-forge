# 01 - The Cold Forge protocol

Eight ordered phases. Each phase has an explicit entry condition, an explicit
exit condition, and a documented failure mode. A run that cannot pass a gate
either resolves the obstacle or aborts. No silent forward motion.

## Phase 0 - Target selection

**Goal:** pick one repo and one issue per run. Avoid picking two issues "in
parallel" until the protocol has been run end-to-end at least once.

**Entry:** an unstructured pool of candidate repos and issues.

**Inputs to weigh:**

- **Repo health.** Stars are not signal. Open issue count, time-to-first-response
  on recent issues, and recent commit cadence are signal.
- **Maintainer engagement on the candidate issue.** A reporter comment + a
  silent maintainer is harder than a reporter comment + a triaging maintainer
  who has labeled it.
- **Issue concreteness.** A reproducible bug with a known trigger is a
  candidate. A vague "feels slow sometimes" is not.
- **Scope.** A fix that fits in under fifty lines, in fewer than three files,
  is much more likely to merge than a refactor.
- **Saturation.** If three external contributors have opened PRs on the issue
  in the last week, the issue is over-served. Pick a different one.

**Exit:** a single `(repo, issue_number)` tuple with a one-sentence rationale.

**Failure mode:** spending more than thirty minutes on Phase 0 means the pool
is wrong. Step back, refresh the candidate pool, do not advance.

## Phase 1 - Dedupe check

**Goal:** confirm no open PR already addresses the issue.

**Mechanical steps:**

```
gh pr list --repo <owner/<repo> \
  --state=all \
  --search "#<issue> in:body,title" \
  --json number,title,state,author
```

Then read every result. A closed PR on the same issue often means the bug is
real but the previous attempt missed something. Read the closing comment before
starting your own attempt.

**Exit:** zero open PRs reference the issue, AND no closed PR exists that was
closed for "this approach is wrong, the right approach is X". If a closed PR
exists with a maintainer comment explaining the right approach, your Phase 4
plan must align with X.

**Failure mode:** an open PR already exists. Read it. If it is stalled and the
author has not responded in 30+ days, comment offering to take over. Do not
silently open a competing PR.

## Phase 2 - Issue triage

**Goal:** read every comment on the target issue, not just the body. Most
contribution failures originate here.

**Read for:**

- **The actual repro steps.** Sometimes the body says "X is broken" and a
  later comment says "actually it only breaks when Y, here is the minimal
  repro." That later comment is the spec.
- **Maintainer responses.** If a maintainer has asked the reporter for more
  info and the reporter has not replied, the issue is in needs-reproduction
  limbo. Pick a different issue.
- **Community fix proposals.** This is the surface where [Phase 4c, the ethical
  gate](03-ethical-gate.md) operates. Any comment containing phrases like
  "happy to submit a PR if so" or "I think the fix is X, let me know if I
  should open it" claims the issue socially even though no PR has been opened.
- **By-design responses.** A maintainer comment explaining "this is intentional,
  not a bug" disqualifies the issue. Do not advance.

**Exit:** a written summary of the bug's actual trigger, the maintainer's
position on it, and any community claim status.

**Failure mode:** the issue body and the comment thread disagree. The thread
wins. If the thread reveals the bug is by-design or already fixed in a later
release, abort and return to Phase 0.

## Phase 3 - Static analysis

**Goal:** locate the root cause in source. Not the symptom, the cause.

**Mechanical steps:**

- `grep` for the visible symptom string (an error message, a UI label).
- `git blame` the responsible file. Recent changes often introduced the bug;
  old code rarely is the culprit unless the bug is platform-specific.
- Read the function calling and the function called. The bug often lives at
  the interface between two correct-on-their-own functions.
- Look for asymmetric patterns. If function A retains state and function B
  does not, and they share an interface, the asymmetry is suspicious.

**Output:** a paragraph identifying the exact line range that causes the bug,
plus one paragraph on WHY that line range causes the bug. The "why" is the
test of root-cause understanding. If you cannot write the why, you do not
understand the fix yet.

**Exit:** root cause identified, written down, defensible.

**Failure mode:** the bug appears to span multiple subsystems with no clean
single cause. Either the diagnosis is wrong (return to Phase 2 and re-read the
thread), or the bug is too large for a drive-by contribution. Abort.

## Phase 4 - Surgical fix

**Goal:** the minimum-blast-radius change that resolves the root cause.

**Constraints:**

- **One conceptual change per PR.** If the fix requires renaming a variable
  AND adding a retention path, the renaming is a separate PR.
- **Match the local style.** Read three nearby functions before writing.
  Copy the indentation, the brace placement, the naming convention. The PR
  must look like the maintainer wrote it.
- **No new abstractions** unless the fix genuinely requires one. Adding a
  helper function that is called once is over-engineering.
- **Preserve invariants.** If the surrounding code has a comment claiming
  "X is always non-null here", do not introduce a path that violates X.
- **Mirror existing patterns** when available. If the same file already
  implements a similar contract correctly elsewhere, the fix should mirror
  that pattern. This makes the diff trivially reviewable.

**Output:** the diff. Plus a sentence on why each changed line changed.

## Phase 4.5 - CLA preflight

**Goal:** verify the project will accept an external PR.

**Mechanical steps:**

- Look for `CONTRIBUTING.md` and read it.
- Look for a CLA bot in recent merged PRs (`CLAassistant`, `cla-bot`, etc).
- If there is a CLA, sign it BEFORE pushing the PR. Some projects auto-block
  merges until CLA is signed.
- Confirm the project accepts external contributions. A small number of
  projects merge only their own employees.

**Exit:** CLA is signed or not required. Project is open to externals.

**Failure mode:** a CLA exists and you cannot or will not sign it. Abort.
Note this in the memory layer so future runs against the same project skip
Phase 0 selection.

## Phase 4c - Ethical gate

The most easily skipped phase and the one whose violation does the most
damage. Detailed in [03-ethical-gate.md](03-ethical-gate.md). Summary:

- Re-scan the issue thread for community members in flight.
- If none, advance.
- If one exists, comment on the issue offering to defer or collaborate,
  with `Co-authored-by` attribution. Wait 12 to 48 hours for response.

## Pre-PR Torture (L1 to L5)

The mechanical validation harness, detailed in
[02-pre-pr-torture.md](02-pre-pr-torture.md). A run that fails any level
puts the protocol in safe-mode rather than pushing the PR.

## Phase 5 - Fork + branch + PR

**Goal:** open the PR with a maintainer-ready body and a clean commit history.

**Mechanical steps:**

```
gh repo fork <owner>/<repo> --clone=false
git remote add fork <fork URL>
git checkout -b <conventional-branch-name>
git add <specific files only, never -A>
git commit -m "<conventional commit message>"
git push -u fork <branch>
gh pr create --repo <upstream> \
  --head <fork-owner>:<branch> \
  --base <main or develop> \
  --title "..." \
  --body "..."
```

**Branch naming:** match the project convention. Look at recent merged PRs.
Common patterns: `fix/<issue>-<slug>`, `fix-<slug>-<area>`, `feat/<slug>`.

**Commit message:** Conventional Commits where the project uses them
(`fix(area): description, close #N`). Plain imperative where not.

**PR body structure:** Summary, Root cause, Fix, Tests, Validation, Notes.
Each section terse but complete. No marketing prose.

**Identity hygiene:** the git author must match the GitHub account opening
the PR. Use a no-reply email tied to that account if the project auto-runs
CLA Assistant.

## Phase 6 - Maintainer signal

**Goal:** read the response window correctly. Avoid two failure modes:
nudging too early (annoying) and waiting too long (PR rots).

**Cadence:**

- First check: 24 hours after PR open. Has anyone labeled it, commented,
  or run CI?
- First nudge: 5 to 7 business days, only if zero human activity. Phrase:
  "Ping on this. CI is green and the diff is small. If there is something
  to rebase, split, or test against, happy to do it. If priority shifted,
  let me know and I can close to keep the board clean."
- Second nudge: 10 to 14 days. Phrase: "Will close if no response by
  <date>. Code is preserved on the branch if it becomes useful later."
- Close: at the second-nudge deadline if still silent. Closes the
  contribution cycle cleanly.

**Failure mode:** silence past two nudges. Do not chase further. Close,
document the outcome in the memory layer, move to the next target.

## Phase 7 - Persistence layer

**Goal:** the next session starts from the position this one ended in, not
from scratch.

**Three artifacts per shipped PR:**

1. A dev-env quirks note for the project (build prerequisites, test runner
   shape, lint rules, branch model, CLA status). See
   [05-rag-memory.md](05-rag-memory.md).
2. An update to the PR scoreboard (state, last signal, monitor cadence).
3. A worked-example writeup at portfolio scope, if the run is portfolio-worthy.

Phase 7 is the only phase that pays compounding interest. Skipping it makes
the next contribution to the same project repeat all the discovery work.
