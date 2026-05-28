# 03 - Ethical gate (Phase 4c)

The protocol step most easily skipped under time pressure, and the one whose
violation does the most reputational damage. The mechanical dedupe in Phase 1
(`gh pr list --search "#N"`) is necessary but not sufficient. Issues can hold
**social claims** that are not yet expressed as open PRs.

## The pattern this catches

A community member reads an issue, does the diagnosis, drafts a fix in a
comment, and ends with:

> "Would this direction be acceptable? Happy to submit a PR if so."

They have not opened a PR because they are waiting for the maintainer to
green-light the direction. The issue is socially claimed even though no
PR exists. Opening your own PR with a technically equivalent fix is a scoop.
Even if your fix is structurally cleaner, the social contract is broken: the
maintainer sees a contributor who jumped a community-member's queue.

This is not a hypothetical. It is a routine failure mode in pre-merge OSS
contribution work, and the maintainer's mental cache of "contributors I can
trust" updates negatively in seconds.

## The gate procedure

After Phase 4 (Surgical fix) has produced a diff but before Phase 5 (Fork +
PR) opens the PR, re-read the target issue thread end-to-end. Specifically
look for:

- **Detailed root-cause analyses** by non-maintainers (more than three
  sentences technical content)
- **Phrases that signal in-flight work without an open PR:**
  - "happy to submit a PR if so"
  - "let me know if I should open one"
  - "I can work on this if no one else is"
  - "draft branch here: <link>"
  - "PR coming soon"
- **Maintainer responses still pending.** A community member who proposed a
  direction and is waiting for "@maintainer would this work?" reply is in
  flight even if their original comment is several days old.
- **The reporter's GitHub timeline.** Reporters often open issue + draft PR in
  parallel. If the reporter is the same person who proposed the fix in the
  comments and they have been active in the same area recently, they are
  likely about to submit.

## The decision tree

**If the thread has no detailed community analysis:** advance to Phase 5. The
ethical gate is clear.

**If a community member is in flight:** do NOT push the PR. Instead:

1. Post a comment on the issue addressed to that contributor and the
   maintainer. Format:

   > "Great analysis [@contributor]. I see your proposed [fix shape]. Are you
   > working on the PR, or would you rather I pick this up? Either way works.
   > If I implement based on your proposal I'll credit you as
   > `Co-authored-by`."

2. Wait 12 to 48 hours. Reasonable contributors reply within a business day
   with either "go ahead" or "I'll submit it tonight".

3. **If they reply "go ahead":** advance to Phase 5 with
   `Co-authored-by: <name> <email>` in the commit trailer.

4. **If they reply "I'm working on it":** gracefully step aside. Reply with
   "great, looking forward to your PR" and move on to the next target.

5. **If silent past 48 hours:** advance to Phase 5, still with the
   `Co-authored-by` trailer.

## Same-comment maintainer mention

The comment from step 1 should also surface any open architectural question
to the maintainer. Example:

> "@maintainer - the proposed direction passes the failing repro test I wrote.
> Before code lands, is the breaking-change shape acceptable, or should we
> route this as a deprecation cycle instead?"

This converts "comment-first" from pure-ethics into pure-progress. The
maintainer gets pinged about the direction, the community member retains
their claim, the protocol gets architectural clearance, and the eventual PR
arrives pre-blessed.

## Anti-patterns

The following actions break the gate and should never happen:

- Opening a PR titled "I implemented @contributor's idea" with no
  `Co-authored-by`. The contributor gets a notification about a PR built on
  their analysis with no attribution.
- Treating "no existing PR linked from the issue" as sufficient dedupe.
  Comments can hold uncommitted-but-claimed work.
- Sliding in with a "better" approach when the community member's proposal
  was already green-lit by a maintainer. Even if the technical fix is
  superior, the social contract is broken.
- Posting the comment from step 1 but pushing the PR before the 12-to-48 hour
  window elapses. The comment is theater if the action does not match.

## Why this is in the protocol

In a well-functioning OSS project, contributor trust is a finite resource
managed by the maintainer. Scooping a community member burns trust faster
than any technical mistake. A technical mistake gets a review comment and a
fix. A social mistake gets the contributor mentally categorized as "watch
this one" for months. The protocol exists in part to make sure that category
never applies.
