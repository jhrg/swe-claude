# Task Plan: Add feature lifecycle tracking (`FEAT-###` log) to the template

**Task:** TASK-001
**Status:** In Progress
**Created:** 2026-11-08

## Summary

Add a fourth ID space, `FEAT-###`, with a log at `ai-docs/features/FEATURE-LOG.md`.
It tracks each accepted feature from `Planned` to `Done` and groups the FR/NFR/UC
IDs and plan that belong to it. Add two commands. `/new-feature` logs a feature.
`/features` shows the board and changes statuses, and on `Done` it marks the
feature's FRs `Implemented`. Wire the log into the existing commands and docs,
following the pattern of `BUG-LOG` and `TASK-LOG`. GitHub-issue linkage via `gh` is
optional.

## Motivation

A project using this template can already answer "what bugs are open?" and "what
tasks are open?" but not "which features are planned, in flight, and done?" Part of
that answer is in FR `Status` (`Proposed / Approved / Implemented`), and part is in
plan `Status` (`Draft … Done`). Neither groups the several FRs that make up one
feature, and a feature that was accepted but has no plan yet doesn't show up
anywhere.

Proposed features (ideas not yet accepted) are deliberately not this repo's
concern. They live wherever a project collects ideas, such as GitHub Issues, Linear,
or meetings. A feature enters the repo when it's accepted, and its entry records
where it came from.

## Related requirements / constraints

None. This repo's `ai-docs/requirements/` and `ai-docs/constraints/` hold only
template placeholders; this task changes the template itself. Two existing rules
shape the design:

- CLAUDE.md "don't invent": a `FEAT` must cite real FR/NFR/UC IDs before it is
  logged. If none exist, `/new-feature` offers `/new-requirement`, as `/plan-feature`
  already does.
- CLAUDE.md "only the user sets `Done`" (from `TASK-LOG` and `BUG-LOG`) applies to
  `FEAT` as well. A `FEAT` reaches `Done` only through a `/features` call the user
  makes.

## Decisions (settled 2026-11-08)

- **Prefix:** `FEAT`.
- **Statuses:** `Planned` → `In Progress` → `Done`, plus `Won't Do` and
  `Superseded by FEAT-0XX`. There is no `Plan Ready` status, because the entry's
  `Plan:` field already shows whether a plan exists.
- **Logging:** a separate `/new-feature` command logs a feature with
  `Status: Planned`, so a feature can be recorded before anyone plans it.
- **Status changes:** go through `/features`.
- **On `Done`:** `/features` marks the feature's FRs `Implemented`.

## Scope

**In scope**, enumerated so completeness can be checked off:

New files:
1. `ai-docs/features/FEATURE-LOG.md`: CONTRACT comment, conventions (ID, the status
   values above, `Done` set only by the user), a `FEAT-EXAMPLE` section, and the
   closing "add new entries via …" comment. This copies the shape of `TASK-LOG.md`.
   Entry fields: Title, Status, Created, Requirements (FR/NFR/UC IDs),
   Constraints (IC IDs), Plan (`None yet` or a path), Origin (URL or `TBD`),
   Summary, and an append-only Status history (date, old → new).
2. `.claude/commands/new-feature.md`: `/new-feature <description>` interviews for
   title, summary, and Origin. It reads the requirement docs and identifies the
   backing FR/NFR/UC IDs. If there are none, it stops and offers
   `/new-requirement`, and logs nothing. It flags `Proposed` FRs as unconfirmed and
   never cites `Deprecated` ones. It assigns the next `FEAT-###` and appends the
   entry with `Status: Planned`, `Plan: None yet`. It doesn't plan.
3. `.claude/commands/features.md`: `/features`
   - **No arguments:** prints the board grouped by status, read-only. Empty groups
     are omitted. Each entry shows its FR IDs, plan link, and Origin.
   - **`/features FEAT-### <status>`:** changes the status. It validates the
     status value (`Superseded` needs a target `FEAT`) and appends a line to Status
     history.
   - **On `Done`:** sets each FR cited by the entry to `Implemented` in
     `functional-requirements.md`, with these exceptions:
     - It skips FRs that are `Deprecated` or `Superseded`.
     - If an FR is also cited by another `FEAT` that isn't `Done`, it asks before
       changing that FR rather than marking it.
     - It reports every FR it changed. NFR and UC statuses are left alone.
   - **GitHub step, optional:** if `Origin` is a `github.com/.../issues/N` URL and
     `gh auth status` succeeds, it *offers* to comment on the issue with the new
     status (and to close it on `Done`). It asks each time, because this is an
     outward-facing action. If `gh` is missing or the Origin isn't an issue, it
     skips silently.

Modified files:
4. `.claude/commands/plan-feature.md`: accepts a `FEAT-###` as the argument. Given a
   description instead, it looks for a matching `FEAT`. If there isn't one, it
   offers `/new-feature` first, the same way it offers `/new-requirement` when FR
   backing is missing. Once the plan is written, it sets the entry's `Plan:` field.
   It doesn't change the `FEAT` status.
5. `ai-plans/_template-plan.md`: add a `**Feature:** FEAT-###` header line.
6. `.claude/commands/trace.md`: look up `FEAT-###` definitions in `FEATURE-LOG.md`,
   report which features cite a given FR/NFR/UC, and flag a `FEAT` with
   `Plan: None yet`. Add `FEAT` to the description line.
7. `.claude/commands/audit-plan.md`: a feature plan is expected to cite a `FEAT-###`.
   Verify that the ID exists and that the entry's FR list matches the plan's
   "Requirements traced" section. Report a missing `FEAT` as a gap, not an error,
   because older plans won't have one.
8. `.claude/commands/init-planning.md`: ask where the project collects proposed
   features (GitHub Issues, another tracker, or nowhere) and record the answer in
   `CLAUDE.md` Conventions so `Origin:` has a known shape.
9. `CLAUDE.md`:
   - Add `FEATURE-LOG.md` to the "Read before planning" table.
   - Add the `FEAT` prefix and a "fourth kind" paragraph to ID conventions.
   - Add `/new-feature` and `/features` to the commands table.
   - Add a one-line definition: FR `Proposed` means "requirement drafted, not
     confirmed" and is different from a proposed feature that lives outside the
     repo.
   - State that FR `Implemented` is set by `/features` when a feature reaches
     `Done`.
10. `CLAUDE-bes.md`: the same edits as item 9. It duplicates those sections.
11. `README.md`: add to the "What you get" tree, add a `FEAT` paragraph after the
    `TASK` paragraph, and add `/new-feature` and `/features` to the command list.
12. `ai-docs/requirements/functional-requirements.md`: on the status line, add a
    note on what `Proposed` means and that `/features` sets `Implemented`.

**Out of scope:**
- `fix-bug.md`, `plan-task.md`, `deep-dive.md`, `new-requirement.md`, and
  `requirements-reviewer.md`. A bug or task *may* mention a feature, but adding a
  `FEAT` field to them is a separate decision.
- `java-conventions.md` and `python-conventions.md`. Their `TASK` mentions are
  examples and need no `FEAT` counterpart.
- `link-swe-claude.sh` needs no change, because it links `ai-docs/` as a whole and
  so picks up `ai-docs/features/`.
- `.claude/settings.json` permissions for `gh`. Each project can add these itself.
- Any GitHub Projects v2 board sync. Linkage covers issues only.
- NFR/UC status changes on `Done`.
- `Claude-outputs/` (historical audit output).
- `ai-docs/tasks/TASK-LOG.md` and `ai-plans/_template-task.md`. They match the grep
  only because they define `TASK` itself.

## Approach

1. Write `FEATURE-LOG.md` (item 1) first. Every other edit refers to its field
   names and status values.
2. Write `/new-feature` (item 2).
3. Update `_template-plan.md` and `plan-feature.md` (items 4 and 5) as one step, since
   the command fills in the template and links the plan back into the entry.
4. Write `/features` (item 3), including the FR update and the GitHub step.
5. Update `trace.md` and `audit-plan.md` (items 6 and 7).
6. Update `init-planning.md` (item 8).
7. Update `CLAUDE.md`, `CLAUDE-bes.md`, `README.md`, and the FR doc note
   (items 9–12).

Completeness check: before editing, record
`grep -rlE 'TASK-|TASK-LOG|plan-task' --exclude-dir=.git --exclude-dir=Claude-outputs .`
(11 files today, listed in the Work log). After editing, each of those files is
either in the in-scope list above with a matching `FEAT` change, or is named in
Out of scope. No file may be in neither list.

## Verification

- Run the completeness grep above and compare its results to the in-scope and
  out-of-scope lists.
- `grep -rn 'FEAT' --exclude-dir=.git .` shows the same status values and field
  names in every file. No file mentions `Plan Ready` for a `FEAT`, and nothing
  drifts between `FEATURE-LOG.md`, `new-feature.md`, `features.md`, and
  `plan-feature.md`.
- `diff CLAUDE.md CLAUDE-bes.md` shows only the differences that were already
  there; the new `FEAT` text is identical in both.
- Dry run in a scratch project linked with `link-swe-claude.sh`:
  1. `/new-requirement` twice, giving FR-001 and FR-002.
  2. `/new-feature`, which creates `FEAT-001` citing both, `Planned`.
  3. `/plan-feature FEAT-001`. The plan is written, the `Plan:` field is set, and
     the status is still `Planned`.
  4. `/features FEAT-001 In Progress`.
  5. `/features FEAT-001 Done`. FR-001 and FR-002 become `Implemented`.
  6. `/trace FEAT-001` and `/audit-plan` on the plan.
- **Shared-FR check:** add `FEAT-002` citing FR-001 and leave it `Planned`. Then
  mark `FEAT-001` `Done`. `/features` asks before changing FR-001.
- **No FR backing:** `/new-feature` and `/plan-feature` both stop and offer
  `/new-requirement`, and no `FEAT` entry is logged.

## Risks

- **Two sources of status.** `FEAT` status and plan `Status` can disagree. The
  mitigation is to define them in CLAUDE.md: `FEAT` status is the feature lifecycle,
  and plan `Status` describes only the plan document.
- **An FR shared between features** could be marked `Implemented` early. The
  ask-first rule in item 3 handles this.
- **`/features` edits `functional-requirements.md`.** This is the first command
  besides `/new-requirement` and `/init-planning` that writes to the requirement
  docs. It changes only the `Status` cell of the cited rows and reports every
  change.
- **Merge conflicts.** `FEATURE-LOG.md` is another single file that's always
  appended to, the same as the existing logs.

## Open questions

- **TASK-001 category:** it's recorded as "Other — template tooling" because the
  category wasn't confirmed. This doesn't block any step.

## Work log

### 2026-11-08 12:58 MST — Plan created

**Prompt:**

> Yes, plan this. The new branch name is good.

(Responding to the proposal in the same session to add an in-repo feature log,
with proposed features kept outside the repo and optional GitHub-issue linkage, as
`TASK-###` on branch `jhrg/feature-log`.)

Created branch `jhrg/feature-log` from `jhrg/develop`. Read the task template,
`TASK-LOG`, `plan-feature`, `plan-task`, `trace`, `audit-plan`, `init-planning`,
`link-swe-claude.sh`, `README`, and the `CLAUDE.md`/`CLAUDE-bes.md` diff. The
completeness grep found 11 files outside `Claude-outputs/`: `audit-plan.md`,
`deep-dive.md`, `plan-task.md`, `trace.md`, `TASK-LOG.md`, `_template-task.md`,
`CLAUDE-bes.md`, `CLAUDE.md`, `java-conventions.md`, `python-conventions.md`, and
`README.md`. Logged TASK-001 and drafted this plan. Nothing has been implemented.

### 2026-11-08 13:22 MST — Open questions resolved

**Prompt:**

> 1. FEAT is the correct prefix
> 2. Plan Ready is redundant. Add Won't Do and Superseded
> 3. Add teh new-feature command
> 4. Call the command `/features` and it should update statuses
> 5. Yes, mark the FR `Implemented` when a feature reached `Done`

Recorded the decisions. Statuses are now `Planned → In Progress → Done` plus
`Won't Do` and `Superseded`. Added `/new-feature` as item 2. `/plan-feature` now sets
only the `Plan:` field. `/features` changes statuses and marks FRs `Implemented` on
`Done`, asking first about FRs shared with a feature that isn't `Done`. Added
verification steps for both. Only the category question is still open. Nothing has
been implemented.

### 2026-11-08 14:52 MST — Implemented items 1–12; static checks pass

**Prompt:**

> Implement the plan @ai-plans/task-feature-log-plan.md

Set the plan and TASK-001 to `In Progress`. Implemented all 12 in-scope items:
- **New files:** `FEATURE-LOG.md`, `new-feature.md`, `features.md`.
- **`plan-feature.md`:** added a new step 1 that resolves the `FEAT` entry, and
  renumbered the other steps.
- **Other edits:** `_template-plan.md`, `trace.md`, `audit-plan.md`,
  `init-planning.md`, `CLAUDE.md`, and the FR doc note.
- **`CLAUDE-bes.md`:** copied the shared block from `CLAUDE.md`.
- **`README.md`:** tree, `FEAT` paragraph, and command list.

Three additions beyond the item text, all following from it:
- `CLAUDE.md` Conventions gained a `{{Feature proposals}}` placeholder, which
  `/init-planning` fills in.
- `CLAUDE-bes.md` got `Feature proposals: TBD`, because the BES answer isn't known.
- The `CLAUDE.md` branch-discipline and template-filling sections now mention
  `FEATURE-LOG` and `Planned`.

Static verification passed:
- The completeness grep's 12 files are each in scope or out of scope; the extra
  one is this plan file.
- `Plan Ready` appears only in `TASK` contexts.
- The shared block in `CLAUDE.md` and `CLAUDE-bes.md` is identical.

**Still to do:** the dry run in a scratch project, including the shared-FR and
no-FR-backing checks. It wasn't run from this session. Nothing is committed yet.
