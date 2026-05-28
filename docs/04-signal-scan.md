# 04 - Signal scan

Reconnaissance methodology for selecting Phase 0 targets. Cold Forge does not
generate target candidates from nothing; it filters an external signal stream
into a ranked queue.

## Three input channels

Three independent channels provide candidates. Cross-correlating them filters
out noise and surfaces signal.

### Channel 1: maintained-by-someone-else issue trackers

For repos already in the candidate pool, refresh the open-issue list and
look for:

- **Newly opened issues with a reproducible bug shape.** Filter by date
  opened, then read the body. Reproducible bugs (with specific steps,
  versions, error messages) are candidates. Vague bugs are not.
- **Issues labeled `good first issue` or `help wanted`** that have not been
  claimed in the comments. These are explicit invitations.
- **Long-open issues with recent activity.** A bug opened a year ago with a
  comment last week often means a new repro surfaced. Read the recent
  activity to confirm.

Mechanical query template:

```
gh issue list --repo <owner>/<repo> \
  --state open \
  --label "bug" \
  --search "sort:created-desc" \
  --limit 50 \
  --json number,title,createdAt,labels,comments
```

### Channel 2: GitHub notifications on existing relationships

If the protocol has produced PRs in the past, GitHub will route reviews,
comments, and mentions to the notification feed. This is the highest-signal
channel:

- A maintainer comment on an open PR is a Phase 6 input.
- A label change ("needs-review" applied) is forward motion.
- A mention from a related PR by another contributor can surface a missed
  area.

Mechanical query template:

```
gh api notifications --jq '
  .[] | {
    repo: .repository.full_name,
    title: .subject.title,
    type: .subject.type,
    reason: .reason,
    updated: .updated_at
  }
'
```

Notifications are processed into action buckets:

- **Reply required:** maintainer asked a question or requested changes.
- **Test failure on PR:** CI flagged a problem requiring re-push.
- **Comment on issue I commented on:** the thread evolved, may need follow-up.
- **PR merged:** Phase 7 work (update memory layer, update portfolio).
- **PR closed without merge:** Phase 6 done, update scoreboard with reason.

### Channel 3: ecosystem proximity

Patterns that surface candidates from outside the active queue:

- **Cluster siblings.** If a fix landed on issue X and the same repo has
  open issues Y and Z with similar root-cause shape, Y and Z are candidates.
- **Upstream-of-upstream.** A fix in repo A might point at a deeper bug in
  the library repo A depends on. The library repo becomes a candidate.
- **Cross-repo failures from CI.** A flaky test in a repo's CI may indicate
  a bug in a shared dependency.

This channel is the lowest-volume but the highest-value. The candidates
surfaced here are non-obvious and over-served much less often.

## Filtering: the saturation matrix

Each candidate from the three channels passes a saturation check before
entering the action queue:

| Signal | Effect |
|--------|--------|
| Recent merged PR count (last 30 days) > 50 | Over-served. Deprioritize. |
| Open external PR count > 30 | Over-served. Deprioritize. |
| Average time-to-first-review > 30 days | Maintainer-throughput-bound. Park unless the fix is critical. |
| CLA bot present and you cannot sign | Disqualified. |
| Repository archived or read-only | Disqualified. |
| No commits in 90 days | Likely unmaintained. Deprioritize. |
| Maintainer responded to a similar PR in the last 14 days | High priority. Active reviewer. |

## The action queue

Filtered candidates land in one of seven buckets:

1. **Ship now.** Issue is concrete, fix is bounded, no community claim,
   maintainer active. Advance immediately to Phase 1.
2. **Investigate first.** Issue is concrete but fix shape unclear. Spend up
   to two hours on Phase 3 reconnaissance before deciding.
3. **Ethical-gated.** Community member in flight on the issue. Comment first
   per [03-ethical-gate.md](03-ethical-gate.md), park 12 to 48 hours.
4. **Monitor only.** Open PR exists from another contributor. Watch for
   their close or merge. No action needed unless they explicitly hand off.
5. **Deferred.** Bug is real but maintainer is slow or unresponsive. Re-check
   in 30 days.
6. **Disqualified.** CLA blocker, archived repo, by-design "bug", or
   over-saturated.
7. **Phase 7 follow-up.** Previous PR has merged; write the worked example
   and update the memory layer.

The queue is reviewed at the start of each session. The top non-blocked item
advances. Items do not skip the queue without justification.

## Handoff to the contribution loop

The signal scan output is **one tuple** for the next Cold Forge run:
`(repo, issue, channel, bucket, rationale)`. The contribution loop reads only
this tuple, not the entire queue. Separation of concerns: scan output is
input to the loop; the loop does not re-scan.

A new scan runs daily or per session, whichever is less frequent. Scanning
more often produces noise; scanning less often misses time-sensitive signals
(a maintainer asking for a small change on a PR that will rot if you don't
reply within a day).
