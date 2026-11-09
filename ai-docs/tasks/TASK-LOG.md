<!-- CONTRACT: scoped engineering/maintenance work that isn't new capability (FR/NFR/
     UC) and isn't corrective (BUG) — dependency bumps, removing dead code or stale
     compile-time directives, retrofitting tests onto code that predates them,
     build/tooling changes. The common thread: the "right" outcome is usually
     mechanical and completeness matters (did every instance get handled?), not
     open-ended design. Delete the TASK-EXAMPLE section once you've seen the shape. -->

# Task Log

Convention: `TASK-###`, sequential, never renumbered. Status: `Open` → `Plan Ready` →
`In Progress` → `Done` (or `Won't Do` / `Superseded by TASK-0XX`). **Only the user sets
`Done`** — it's a claim about verified work, not something to self-report from having
written a plan. Category is one of: Dependency
Update, Code Cleanup, Test Coverage, Build/Tooling, Documentation (add categories as
needed).

---

## TASK-EXAMPLE — Add unit tests for the legacy `parser/` module

*(delete this section once you've seen the shape)*

**Category:** Test Coverage
**Status:** Plan Ready
**Created:** YYYY-MM-DD
**Related requirements/constraints:** None found — `parser/` predates FR/NFR/UC
tracking in this repo
**Plan:** `ai-plans/task-parser-test-coverage-plan.md`

**Scope:** Every public function in `parser/`, tested against the fixtures already
in `parser/testdata/`. Does not include refactoring the module itself, even where
the tests reveal awkward internal structure — that's a separate task if pursued.

**Motivation:** `parser/` has zero test coverage and has caused two regressions in
the last quarter that tests would have caught before merge.

---

## TASK-001 — Add feature lifecycle tracking (`FEAT-###` log) to the template

**Category:** Other — template tooling (category not confirmed with user)
**Status:** In Progress
**Created:** 2026-11-08
**Related requirements/constraints:** None found — this repo's requirement docs are
template placeholders; the work changes the template itself
**Plan:** `ai-plans/task-feature-log-plan.md`

**Scope:** Add a `FEAT-###` ID space and `ai-docs/features/FEATURE-LOG.md` that track
features from `Planned` through `Done`; add `/new-feature` and `/features` commands
(the latter changes statuses and marks FRs `Implemented` on `Done`); wire it into
`/plan-feature`, `/trace`, `/audit-plan`, `/init-planning`, `_template-plan.md`,
`CLAUDE.md`, `CLAUDE-bes.md`, and `README.md`. Optional GitHub-issue linkage via `gh`.
Proposed (not-yet-accepted) features are explicitly out of scope — they live outside
the repo, and a `FEAT` entry only records where one came from (`Origin:`).

**Motivation:** Projects using this template can track bugs and tasks but have no
single place answering "what features are planned, in flight, and done?" FR status
and plan status each hold part of that, and neither groups the FRs that make up one
feature.

---

<!-- Add new tasks via /plan-task, or by hand — keep the section format: metadata
     lines, then Scope / Motivation. Unlike a bug, a task doesn't need a symptom or
     reproduction — it needs a clear boundary of what's included and what isn't,
     since "remove all the X" quietly becoming "remove most of the X" is the
     characteristic failure mode here. -->
