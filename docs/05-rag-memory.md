# 05 - The memory layer

The persistence mechanism that makes the second contribution to a project
cheaper than the first. Two tiers, picked by content shape.

## Two-tier design

After running the protocol for some time it becomes obvious that "memory"
is not one thing. Some content needs to be in the agent's reflexive
context from the first message of every session (rules, conventions,
per-project build quirks). Other content is high-volume operational
state (PR iterations, CI chains, signal scans) that the agent looks up
on demand. Forcing both into the same storage either bloats the start-up
context or makes rules invisible.

The protocol therefore uses two stores:

| Tier | What goes in | Implementation |
|------|--------------|----------------|
| **Auto-load** | Meta-rules + per-project dev-env quirks + protocol docs themselves | Plain markdown files in a known directory. Agent loads the index on session start. |
| **On-demand RAG** | Runtime state: PR scoreboards, CI iteration chains, signal scans, maintainer comments, outreach handoffs, worked-example diagnoses | Embedded vector store with a `memory_save(note, tag)` / `memory_search(query, k)` API, runnable locally. |

The split is by content cadence, not by importance. Both are essential.

## Why not one tier?

Three options that were tried and rejected:

- **An LLM's session context only.** Resets every session. Zero
  persistence across multi-day work. Rejected.
- **Plain markdown only, even for state.** Tried first. Works but breaks
  at volume: hundreds of `wave-N-*.md` and `signal-scan-*.md` files
  clutter the directory, the index file balloons past the auto-load
  truncation limit, and search devolves to grep. Rejected for state.
- **Embedded RAG only, even for rules.** Rejected because rules need to
  fire from the first agent message of a session (e.g., "never use em-
  dashes in PR bodies"). A rule that requires a search round-trip before
  it fires is a rule that gets violated on round one.

Plain markdown wins for rules + low-volume per-project context. Embedded
RAG wins for high-volume operational state.

## Structure

```
memory/
├── index.md              ← one-line pointers to every memory file
├── feedback_*.md         ← rules learned from corrections (durable)
├── project_*.md          ← ongoing project state (refresh often)
├── reference_*.md        ← pointers to external systems
├── <repo>_dev_env_quirks.md   ← per-project build/test/lint knowledge
└── worked_example_*.md   ← case studies of past contributions
```

Files are flat (no subdirectories) so that the index file can point
directly. The index is loaded into every LLM agent session so the agent
can decide which files to read based on the current task.

## File format

Each memory file is a single markdown document with YAML frontmatter:

```markdown
---
name: short-kebab-case-slug
description: one-line summary used by the index to decide relevance
metadata:
  type: feedback | project | reference | dev-env-quirks | worked-example
---

Body content. Linked memories appear as [[other-name]]. Code blocks where
useful. Tables where structured. Plain prose otherwise.
```

The `name` field is what the index links to. The `description` is what the
agent reads to decide whether to load the file. The body is what the agent
reads after deciding.

## What goes in memory

The four categories worth persisting:

1. **Feedback rules.** A correction the protocol learned from a past failure
   ("do not nudge maintainers before five business days"). Durable across
   projects.
2. **Project state.** Snapshot of in-flight PRs, monitor cadence,
   follow-up debts. Refreshed every time the state changes.
3. **External references.** "Issue tracker for project X lives at <URL>" or
   "build instructions for project Y are in `doc/BUILDING.md`". Pointers,
   not content.
4. **Per-project dev-env quirks.** Build prerequisites (Rust toolchain,
   wasm-pack), test runner shape (custom otest vs jest), lint setup
   (oxfmt vs prettier vs biome), branch model (master vs main vs develop),
   CLA status (required or not, who signed), known build-pipeline bugs
   (Windows path escape issues). The single highest-leverage category:
   without these notes, the second session against a project repeats the
   first session's discovery work.

## What does NOT go in memory

- **Code patterns or architecture of the upstream project.** The source is
  authoritative. Memory pointers go stale.
- **Git history facts.** `git log` is authoritative.
- **Debugging fix recipes.** The fix is in the code; the commit message has
  the context.
- **Anything already in `CONTRIBUTING.md` of the target project.** Re-reading
  the source is faster than re-reading a copy.
- **Ephemeral state.** Current session task list, in-progress notes, scratch
  thinking. These belong in the session itself, not in persistent memory.

## The two-step save

Saving a memory is mechanical:

1. Write the file to its own path with the frontmatter.
2. Add a one-line pointer to the index file.

The index is the only file an agent loads on every session. Individual
memories are loaded on-demand based on the description match.

## Index format

```markdown
# Memory index

- [Feedback: never nudge before five days](feedback_nudge_cadence.md) -
  short summary of when this applies
- [Project X dev-env quirks](projectx_dev_env_quirks.md) -
  npm-based, custom test runner, Rust toolchain needed for full build
- [Worked example: project Y PR #1234](worked_example_y_1234.md) -
  GC race fix, retention pattern from sibling function
```

Each line is under 150 characters. The index is loaded into every session
so the agent has a map of what is available without reading 50 files.

## Staleness handling

Memory files go stale. A `project_state` file describing "PR X is open
awaiting review" becomes wrong once the PR merges. A `dev_env_quirks` file
becomes wrong when the project switches build systems.

Convention: **trust observed reality over memory.** If a memory says "the
test runner is X" and the current `package.json` says "the test runner is
Y", trust the file. Update the memory.

Before recommending an action based on memory content, verify the memory
is still current. The rule of thumb: memory is a hypothesis, the source
code is the fact.

## Why this is in the protocol

Knowledge loss between sessions is the most expensive failure mode of
contributing across many projects. Each project has its own build pipeline,
test framework, lint setup, branch model, and submission etiquette. Without
a memory layer, every session re-discovers all of it.

A memory layer that captures even a quarter of this knowledge per session
makes the protocol's marginal cost-per-PR decline over time. A pipeline
that does not learn does not scale.
