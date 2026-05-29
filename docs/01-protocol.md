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

## Phase 4d - Humanization scrub

**Goal:** strip every AI-tell from text that will leave the contributor's
context (PR body, PR comments, commit messages, cold mail bodies, portfolio
prose). LLM-generated prose has telltales that read as machine-written to
humans on GitHub and in cold mail; they undermine the builder-to-builder
framing the protocol depends on.

**Mechanical sweep:**

1. **No em-dash, no en-dash.** Grep for the two unicode codepoints
   `U+2014` and `U+2013`. Replace with periods (start a new sentence),
   commas (continue the clause), or parentheses (parenthetical). Range
   constructs become "X to Y", not the en-dash form. A regular hyphen
   (`U+002D`, `-`) is fine in compound modifiers like `breaking-change`.
2. **No LLM-marker vocabulary.** Run the AI-tell phrase list against
   every text artifact about to leave your context. Common offenders:
   - Process verbs: `delve into`, `dive into`, `unpack`, `distill`,
     `navigate`, `harness` (when not naming an actual test harness),
     `leverage` (when "use" or "rides" works), `streamline`, `empower`
   - Marketing register: `robust`, `comprehensive`, `seamless`,
     `transformative`, `revolutionary`, `elegant solution`,
     `cutting-edge`, `game-changing`, `meticulous`, `quintessential`,
     `invaluable`, `catalyst`, `paradigm`
   - Hedge filler: `It's worth noting`, `It's important to note`,
     `crucially`, `significantly`, `in essence`, `ultimately`,
     `Additionally,`, `Moreover,`, `Furthermore,`, `In conclusion,`
   - Metaphor cliché: `testament to`, `tapestry`, `bedrock`,
     `ever-evolving`, `in the realm of`, `unwavering`, `unique blend`,
     `plethora`, `myriad`, `nuanced`
   - AI exuberance: `amazing`, `awesome`, `fantastic`, `wonderful`,
     `Hello there!`, `Great question!`, `Sounds good!` with exclamation
   - AI self-reference: `As an AI`, `As a language model`, `I cannot`
     in any context that suggests a refusal, `I hope this helps`,
     `Let me know if`, `Feel free to`, `Remember to`
3. **Allow technical terms that match the AI-tell substring but are not
   AI-tell in context.** Examples:
   - `harness` in `Pre-PR Torture harness` (the protocol's own term)
   - `leverage` in `high-leverage` describing a real signal
   - `bedrock` when naming AWS Bedrock or a literal foundation
   - `paradigm` when contrasting two architectural paradigms in code
   Decide by reading the line. If a human engineer would write the same
   thing, it stays; if it reads like a marketing brochure, rewrite.
4. **Italian prose specifically.** Avoid English-loaded constructions
   ("permettere di", "consentire di" used as filler), keep Italian
   colloquialisms where the audience is Italian-native. Em-dashes do
   not exist in standard Italian typography either, so the U+2014 ban
   applies identically.

**Exit:** the text reads as something a real engineer would type at
00:30 after a long day. Not as something a model would write in batch.

**Failure mode:** the sweep finds zero AI-tells, but the text still
reads generic. Cause: too many `<sub-clauses, that, when, parsed,
read, like, a, machine>`. Rewrite into shorter declarative sentences.
Or, the sweep finds dozens of AI-tells in a private draft: trace the
provenance (which session generated it, with which prompt) and update
the upstream prompt so the next generation comes clean.

**This sweep is mandatory before any of:** PR body push, PR comment
post, commit message commit, cold mail send, portfolio commit push,
public README commit push.

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

**Goal:** read the response window correctly. Avoid three failure modes:
nudging too early (annoying), waiting too long while the PR rots, and
making the maintainer chase CI failures you could have fixed yourself.

**Cadence with humans:**

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

**Failure mode (human channel):** silence past two nudges. Do not chase
further. Close, document the outcome in the memory layer, move to the
next target.

### Phase 6b - Continuous CI iteration

A subphase activated automatically after Phase 5 push and running in
parallel with human-signal monitoring. The maintainer's job is to review
your **content**, not to chase your CI failures. Every mechanical CI
failure (compile error, test failure, lint warning, naming convention,
formatting drift) you fix yourself before they have to look at it.

**Loop until terminal state:**

1. **Trigger.** Every push to the PR branch auto-triggers CI on most
   projects. First-time external contributors may be gated on
   `action_required` (a maintainer must click "Approve and run" once).
   That gate is the only thing you cannot resolve without the human.
2. **Poll.** While at least one CI job is `in_progress`, poll the status
   every **180 seconds** (3 minutes). Cache stays warm, runner load
   stays low. When all jobs reach a terminal state (success or failure),
   stop polling.
3. **Triage per failed job.** For each `failure` conclusion:
   - Pull the failed log: `gh run view <run-id> --log-failed`
   - Identify the root cause line. The first `error:` or `FAILED` line in
     the log is usually it.
   - Classify the failure:
     - **Compile error:** mechanical, fix immediately.
     - **Test failure** caused by your diff: mechanical, fix immediately.
     - **Lint / format / naming convention warning:** mechanical, fix
       immediately (e.g., clang-tidy `readability-identifier-naming`,
       eslint, oxfmt, biome, gofmt).
     - **Flaky test** unrelated to your diff: do not chase. Add a note in
       the PR comment thread once. The maintainer decides.
     - **Pre-existing failure on main:** do not chase. Mention in PR
       comment with a link to the same job failing on `main`.
4. **Surgical fix.** Same shape as Phase 4 (one conceptual change,
   minimal blast radius). The fix MUST address the exact line the log
   pointed at. No drive-by cleanups.
5. **Push.** New commit, dedicated message (`test: rename X to Y (clang-tidy
   naming)`, `fix(area): handle null in Y (CI: compile error)`, etc).
   Each iteration is one commit. Resist squashing in flight; readable
   iteration history is what shows the maintainer you debugged carefully.
6. **Repeat from step 1** until all CI jobs are `success` OR until the
   only remaining failure is one of:
   - A flaky test (documented + accepted),
   - A pre-existing main failure (documented + linked),
   - An `action_required` gate awaiting maintainer approval (waiting).

**Polling cadence rules:**

- `in_progress` jobs: poll every **180s** (3 min). Cache stays warm
  through the 5-minute prompt cache window.
- `action_required` (no in_progress, no movement): poll every **1200s**
  (20 min). Maintainer approval can take hours; tight polling is waste.
- All terminal: stop polling. Act on result.

**The action_required trap (critical):**

For first-time external contributors, GitHub Actions gates every push on
manual maintainer approval. The workflow run reports `status: completed`
and `conclusion: action_required`. This is **not green**. It is "pending
human, paused before execution". A monitor script that checks for
"completed AND not failure" will misclassify this as success. The
correct classification:

```
status      conclusion        meaning
--------    ---------------   --------------------------------------
queued      (none)            scheduled, not yet picked up
in_progress (none)            executing now; keep polling 180s
completed   success           green; record + advance
completed   failure           red; trigger triage + fix loop
completed   action_required   paused awaiting human approval; do not
                              treat as success, do not push more
                              commits, just wait + poll 1200s
completed   neutral, skipped  ignore (not informative)
completed   cancelled         someone aborted; investigate why before
                              re-running
```

After the maintainer approves once, subsequent pushes may or may not
re-gate depending on repo policy (`First-time contributors` vs `Require
approval for all outside collaborators`). When in doubt, assume each
new commit re-gates. This caps the iteration speed to roughly the
maintainer's reply latency on the approval click, not the CI runtime.

**Iteration limit:**

- If 5 CI iterations in a single session do not converge to green, STOP.
  Open a comment on the PR with the state, the last log excerpt, and the
  question. Maintainers prefer "I am stuck on X, here is what I have
  tried" over "I keep pushing variations".
- If a CI failure requires architectural input (e.g., test fixture
  redesign), do not iterate alone. One comment, then wait.

**Documentation requirement (feeds Phase 7):**

Every iteration cycle (push → fail → fix → push) is logged to the RAG
with the diagnosis chain. The next session inherits the iteration history
without re-reading the CI logs.

## Phase 7 - Persistence layer

**Goal:** the next session starts from the position this one ended in, not
from scratch.

Persistence is the only phase that pays compounding interest. Skipping it
makes the next contribution to the same project repeat all the discovery
work; skipping it within a long-running PR means the next agent that picks
up the session re-reads the CI logs from scratch.

### Storage selection

Two storage tiers, picked by content shape:

- **Embedded RAG** (semantic, cross-session, cross-project): runtime state,
  signals, merges, CI iteration chains, scoreboard deltas, worked examples,
  outreach handoff status. Use a `memory_save`-style API (in the reference
  implementation: `mcp__local-offload__memory_save(note, tag)`).
- **Plain markdown files** (auto-loaded into agent context): meta-rules
  (no-em-dash, no-new-fronts-during-pending), per-project dev-env quirks
  (build prerequisites, test runner shape, lint rules, branch model, CLA
  status), the protocol docs themselves. See
  [05-rag-memory.md](05-rag-memory.md).

The split matters: rules need to be in the agent's reflexive context from
the first message of every session. State needs to be retrievable on
demand but does not need to bloat every session's working memory.

### Save cadence (RAG)

Save on every signal change, not just at PR close:

| Event | Tag | Body |
|-------|-----|------|
| PR pushed | `wave-N-pr` | branch, diff summary, PR URL, hypothesis being tested |
| Maintainer comment received | `wave-N-pr` | who, when, verbatim or paraphrase, action implied |
| CI run completed (success or failure) | `wave-N-pr` | which workflows, result, failure logs grep'd if any |
| CI iteration (push fix → re-run) | `wave-N-pr` | what the previous failure was, what the fix changes, what to watch on next run |
| PR merged | `wave-N-merge` | maintainer, time, diff size, turnaround, maintainer-specific notes (last day, holiday, etc) |
| PR closed without merge | `wave-N-closed` | reason, maintainer wording, follow-up path if any |
| Cold mail sent / replied | `outreach-<org>` | recipient, subject, key contents, next checkpoint date |
| Dev-env quirk discovered | `dev-env-<repo>` (.md, not RAG) | so future sessions auto-load it |
| Worked example complete | `wave-N-example-<slug>` | full diagnostic chain for portfolio + future RAG retrieval |

### Save shape (RAG)

Each note must be **self-contained**. A fresh agent reading the note in a
later session has no implicit context. Include:

- Absolute dates (not "today", not "yesterday"). The note will be read in
  a future where those words point at different days.
- Real names and handles (not "the maintainer", not "him").
- URLs to PRs, issues, comments (not just numbers).
- Why the action was taken, not just what was done.
- What is known vs what is assumed.

### Save cadence (markdown)

Lower frequency. Update when:

- A new meta-rule is learned and crystallized (write a `feedback_<rule>.md`).
- A new project enters the candidate pool and dev-env quirks surface
  during the first contribution (write a `<repo>_dev_env_quirks.md`).
- A new protocol phase is added or an existing phase is materially
  changed (update the relevant `docs/0N-*.md`).

### Three artifacts per shipped PR

In addition to the streaming RAG saves above, a fully closed PR (merged
or final-closed) produces:

1. A dev-env quirks note for the project, kept as `.md` (auto-load).
2. An update to the PR scoreboard, kept as RAG (`wave-N-scoreboard`).
3. A worked-example writeup at portfolio scope, kept as RAG (`wave-N-
   example-<slug>`) and optionally as `examples/<slug>.md` in the
   public Cold Forge repo if the run demonstrates a teachable protocol
   step.

## Phase 7b - Portfolio update procedure

**Goal:** the public portfolio reflects current state within one commit
of any merge, new senior-tier PR, or material signal change. Stale
numbers are worse than no numbers because they signal carelessness.

**Triggered by any of:**

- PR merged upstream (any repo, any tier)
- PR closed without merge (capacity, by-design, etc)
- New PR opened that is senior-tier or otherwise material
- Maintainer engagement signal worth citing (a "infuriating bug confirmed"
  quote, a code-owner escalation, a clearance from a core maintainer)
- Cold-mail conversion event (call scheduled, contract opened)

**Mechanical steps:**

1. **Update the global counters first.** Every number that appears more
   than once must be updated everywhere. Specifically:
   - Hero meta tile (`hero.meta_upstream` translation key in all
     supported locales)
   - Hero CTA button (`hero.cta_outreach`)
   - Nav badge
   - Outreach section title (`outreach.title`)
   - Section subtitle prose (`outreach.sub`, all locales)
   - `<meta name="description">` for SEO
   - `<meta property="og:description">` for social cards
   The counters to keep coherent: total PRs shipped, merged count, open
   count, closed-for-capacity count.

2. **Update the affected card.** Each upstream PR has a dedicated card
   in `.projects-grid`. On state change:
   - `OPEN` to `MERGED`: swap `.project-badge.public` to
     `.project-badge.merged`, add `.merged-card` class to the article,
     add `<div class="merged-stamp">MERGED</div>` ribbon at the top of
     the article, update the rank line to include `MERGED` and the
     turnaround time
   - `OPEN` to `CLOSED-capacity`: swap badge, add explanatory note in
     the body about why (not-merit, capacity, etc), with the maintainer
     quote if available
   - Senior-tier PR: add `.senior-tier` class + `<div class="tier-stamp">SENIOR</div>`

3. **Insert a new card if the PR is new.** Match the existing card
   template exactly:
   - `<article class="project featured reveal" data-tilt>`
   - `<div class="project-rank">#N · <repo> · PR #X · <one-line> · <maintainer signal></div>`
   - `<div class="project-badge public">…</div>` (or merged variant)
   - `<h3 class="project-title">…</h3>`
   - `<p class="project-claim">…</p>` (italic, one-line headline)
   - `<p class="project-body">…</p>` (root cause + fix + tests)
   - `<ul class="project-tech">` with 6 chips (language, framework,
     lint tool, test framework, LOC counter, test count)
   - `<div class="project-links">` with PR, branch, issue links
   Insert at the top of the chronologically-newest block (cards are
   reverse-chronological after the initial Wave 1 block).

4. **Update the merge milestone banner above the grid.** Format:
   `N/Total` with a gradient text, milestone description, and a CTA
   to the strongest current PR.

5. **Update internationalization parity.** Every English string update
   above must have a matching IT and ZH update. The translation
   convention: keep handles, file paths, code identifiers, commit
   hashes, and PR numbers verbatim. Translate only natural-language
   prose. Em-dash ban applies to all locales identically.

6. **Run Phase 4d humanization sweep on the entire diff.** A merge is
   not an excuse to ship marketing prose. Em-dash count must remain
   zero across `index.html`, `assets/js/i18n.js`, `assets/css/style.css`,
   and every `outreach/*.html` file.

7. **Verify card count parity.** Count `project-rank` occurrences in
   the outreach grid. The number must match the announced count in the
   section title and the hero tile. A card without a counter bump (or
   vice versa) is a portfolio bug.

8. **Mobile breakpoint check.** Visual additions (stamps, banners,
   gradient text) need a fallback below 520px:
   - Stamps: shrink padding, reduce font-size to .6rem
   - Project-rank: cap max-width, add padding-right to clear the badge
   - Banner: grid-template-columns may need to collapse
   Verify via DevTools or a real device.

9. **Commit + push to the portfolio repo.** Commit message states what
   changed and why in one sentence, with optional second sentence for
   context. Em-dash ban applies to commit messages too.

10. **Save the delta to RAG** (`portfolio-state` tag) so the next
    session inherits the current counter and card sequence without
    re-reading the HTML.

**Failure modes:**

- Forgot to bump one of the eight counter sites. Result: hero says 15,
  section title says 16. Fix: add the missing site, re-commit.
- Inserted a new card but did not run Phase 4d on the prose. Result:
  one card contains an em-dash or LLM-tell verb. Fix: scrub, re-commit.
- Did not update the i18n locales. Result: IT user sees stale numbers.
  Fix: sweep `assets/js/i18n.js` for every literal that contains a
  number that the English version changed.
- Mobile layout breaks because the new stamp ribbon overlaps the
  badge. Fix: add the @520px breakpoint for the new element.

**Exit:** all eight counter sites coherent, new card matches template,
i18n EN/IT/ZH parity, Phase 4d clean, mobile breakpoint covered, commit
message clean, RAG note saved.
