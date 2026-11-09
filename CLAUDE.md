# CLAUDE.md — `opendap-registry`

**The OPeNDAP Server Registry.**

**What:** a registry of OPeNDAP *servers*, with emphasis on the
authenticated/access-controlled ones ordinary crawling cannot find.

**Produces:** `data/registry.db` and its exported `servers.json` contract — 357
servers / 313 hosts / 131 institutions.

This repo plans work from written requirements, not from conversation memory or
assumption. Requirements, use cases, and constraints live in `ai-docs/`. Plans that
Claude produces live in `ai-plans/`. Every plan traces back to written IDs: a feature
plan cites FR/NFR/UC/IC IDs, a bugfix plan cites a `BUG-###`, a task plan cites a
`TASK-###` (bugfix and task plans may legitimately cite no FR/NFR/UC). If a *feature*
request has no requirement backing it, that's a signal to add one — not to infer one
silently.

## Read before planning anything

| File / folder | Holds |
|---|---|
| `ai-docs/requirements/functional-requirements.md` | What the system must do — `FR-###` |
| `ai-docs/requirements/non-functional-requirements.md` | Quality attributes & targets — `NFR-###` |
| `ai-docs/requirements/use-cases.md` | Actor-driven flows — `UC-###` |
| `ai-docs/constraints/implementation-constraints.md` | Hard boundaries on *how* — `IC-###` |
| `ai-docs/decisions/DECISIONS.md` | Why past calls were made — `ADR-###` |
| `ai-docs/bugs/BUG-LOG.md` | Deviations from intended behavior, report through root cause — `BUG-###` |
| `ai-docs/tasks/TASK-LOG.md` | Scoped maintenance/engineering work not tied to a feature or bug — `TASK-###` |
| `ai-docs/features/FEATURE-LOG.md` | Accepted features, `Planned` through `Done`, each grouping its FR/NFR/UC IDs and plan — `FEAT-###` |
| `ai-docs/deep-dives/` | Persisted findings from code investigations, one file per question |
| `ai-plans/` | Output: one plan file per feature or bug fix, each citing the IDs above |

Read the requirement and constraint docs **in full** before drafting or revising a
plan — don't rely on what they said earlier in the session, they may have changed.
Also check `ai-docs/decisions/DECISIONS.md` for any accepted ADR the design would
contradict (flag it the way you'd flag an IC tension), and look in `ai-docs/deep-dives/`
for an existing note on the area before investigating it again.

- Ignore `_EXAMPLE_` / `*-EXAMPLE` placeholder entries — they show the format; they
  are not requirements.
- Don't plan against a `Deprecated` entry; follow `Superseded by` to its replacement.
  Call out any `Proposed` entry as not yet confirmed rather than treating it as settled.
- An FR/NFR with status `Proposed` is a requirement that has been drafted but not
  confirmed. It isn't a *proposed feature*: those are ideas not yet accepted, and
  they live outside this repo (see Conventions). A feature enters `FEATURE-LOG.md`
  only once it's accepted.

## ID conventions

- IDs are `PREFIX-###`, zero-padded to 3 digits, assigned sequentially, **never reused
  or renumbered**. If a requirement is dropped, mark it `Status: Superseded by FR-0XX`
  or `Status: Deprecated` — don't delete the row. History is part of why traceability
  works.
- Prefixes: `FR` functional requirement, `NFR` non-functional requirement, `UC` use
  case, `IC` implementation constraint, `ADR` architecture/design decision, `BUG` a
  logged deviation from intended behavior, `TASK` a scoped maintenance/engineering
  task that is neither new capability nor a correction, `FEAT` an accepted feature
  tracked through its lifecycle.
- Every plan phase or major step should cite the ID(s) it satisfies, e.g. `(FR-012,
  UC-003)`. A step with no citation is either scope creep or an uncaptured
  requirement — flag it, don't quietly do it.
- `BUG-###` is a different kind of ID from the rest: it doesn't describe intended
  behavior, it describes a *violation* of it (against an existing FR/NFR/UC where one
  exists, or against an undocumented assumption when it doesn't). Don't force a bug
  into a feature plan's shape — see `ai-plans/_template-bugfix.md`.
- `TASK-###` is a third kind of ID: neither new capability like an FR/NFR/UC nor a
  correction like a BUG — scoped engineering work (dependency bumps, removing dead
  code or stale compile-time directives, retrofitting tests onto code that predates
  them) that doesn't need requirement backing to be legitimate. Don't force it into
  a feature or bugfix plan's shape — see `ai-plans/_template-task.md`.
- `FEAT-###` is a fourth kind of ID: it doesn't state behavior itself, it groups the
  FR/NFR/UC IDs that one feature delivers, links the plan that builds it, and
  tracks its status: `Planned` → `In Progress` → `Done`, or `Won't Do` /
  `Superseded by FEAT-0XX`. Log one with `/new-feature`, only when real
  requirements back it. Change its status only with `/features`, and only `Done`
  when the user says so. On `Done`, `/features` sets the cited FRs to
  `Implemented`; that's the normal way an FR becomes `Implemented`. A feature's
  status is its lifecycle; the plan's own `Status` describes only the plan document.

## The core discipline: don't invent

- If you can't find a requirement backing a feature request, say so and offer to run
  `/new-requirement` rather than writing a plausible-sounding requirement yourself.
  A plan built on an invented requirement looks identical to one built on a real one
  until it's too late to matter.
- If a field is unknown during an interview (priority, target metric, actor), leave
  it as `TBD` rather than guessing. A gap is honest; a plausible guess reads as fact
  later.
- If a proposed plan would violate an implementation constraint (`IC-###`), **say so
  explicitly** and ask how to proceed. Don't silently comply with the constraint by
  redesigning around it without flagging the tension, and don't silently ignore it.

## Filling in templates and logs

When you create a plan or log entry from a template: set `Status` to the single
starting value (`Draft` for plans, `Open` for bug and task log entries, `Planned` for
feature log entries) — never copy the `A | B | C`
option list; set `Created` / dates to today's actual date (run `date +%F` if you
don't have it — never guess); and leave genuinely unknown fields `TBD`.

## Work logs on plans

Every plan in `ai-plans/` ends with an append-only `## Work log` section (see the
templates). The first entry, written when the plan is created, quotes the prompt
that initiated it verbatim. After that, **whenever you do work on a plan** —
revising it, investigating for it, implementing or verifying a phase, changing its
status — append an entry before ending the session's work on it: a heading stamped
with the output of `date '+%F %H:%M %Z'` (run it — never guess the time) and a short
summary of what was done and what's left. Don't edit or reorder earlier entries.

## Branch & working-tree discipline

Before making any change for a plan, task, bug, or feature — a new plan file, a
BUG-LOG, TASK-LOG, or FEATURE-LOG entry or status change, a work-log entry, or code — the working tree must be clean and the
change must happen on its own branch, never directly on `main`. This applies
uniformly to docs and code: drafting a plan, logging a bug/task, and implementing
one all happen on that item's branch.

- **Check first:** run `git status` before touching anything. If there are
  uncommitted edits, stop and ask the user whether to commit, stash, or discard
  them — don't layer new work on top of an already-dirty tree.
- **One branch per item:** before the first edit, confirm whether an existing
  branch already covers this item or a new one is needed. Name new branches
  `jhrg/<slug>`, matching the existing convention (e.g. `jhrg/sd-debug-log`) — a
  short kebab-case description of the plan/task/bug, not the raw ID.
- **Always ask before creating or switching branches** and confirm the name with
  the user first, even when the tree is clean — creating/switching branches
  changes shared git state, so it's never done silently.

## Custom commands available in this repo

| Command | Does |
|---|---|
| `/init-planning` | First-time setup: interviews you and seeds the requirement docs for an existing or new codebase |
| `/new-feature <name or description>` | Logs an accepted feature as `FEAT-###` (`Planned`), citing the FR/NFR/UC IDs that back it |
| `/plan-feature <FEAT-### or description>` | Reads the requirement docs, drafts a phased implementation plan in `ai-plans/`, citing IDs throughout, and links it from the `FEAT` entry |
| `/features [FEAT-### <status>]` | Shows the feature board by status; with arguments, changes a feature's status (on `Done`, marks its FRs `Implemented`) |
| `/fix-bug <description or BUG-###>` | Logs a bug (or resumes one), investigates root cause read-only, drafts a bugfix plan |
| `/deep-dive <area or question>` | Read-only investigation of existing code; checks it against the requirement docs and saves findings |
| `/new-requirement` | Interviews you to add a new FR / NFR / UC / IC entry with the next sequential ID |
| `/plan-task <description or TASK-###>` | Logs a maintenance/engineering task (or resumes one) and drafts a plan for it |
| `/trace <ID>` | Reports everywhere an ID is referenced — docs, plans, code — and flags orphaned requirements |
| `/audit-plan <plan file>` | Checks a plan's ID citations against the requirement docs; reports gaps and coverage, changes nothing |

The `requirements-reviewer` subagent (`.claude/agents/`) reviews the requirement and
constraint docs for vague, untestable, conflicting, or dangling entries. Use it when
asked to review or tighten those docs — it reports findings and never edits.

## Conventions

{{CONVENTIONS — e.g. language/stack, branching model, where code actually lives, test
command, anything a fresh Claude session needs to stop guessing}}

**Feature proposals:** {{where proposed features are collected before they're
accepted — e.g. GitHub Issues in OWNER/REPO, or "nowhere in particular" — so a
`FEAT` entry's `Origin:` has a known shape}}

## Plan mode

For anything nontrivial, prefer Claude Code's built-in plan mode (`Shift+Tab` twice)
combined with `/plan-feature` — read the requirements, think through the approach, and
present the plan before touching code. Read-only research and planning shouldn't need
write access to the codebase.
