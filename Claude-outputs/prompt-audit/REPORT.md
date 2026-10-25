# Prompt audit — claude-swe-planning-template

**Date:** 2026-10-01 · **Scope:** `CLAUDE.md`, the 8 commands in `.claude/commands/`, `.claude/agents/requirements-reviewer.md`, the plan/log templates, the `docs/` contract comments, `README.md`, and `arduino-platformio-conventions.md` (a CLAUDE.md fragment).
**Method:** read every file in full; cross-checked each command's step numbering, ID/status vocabulary, and file references against the docs they operate on. Nothing was run against a real project, so findings about behavior are from reading, not from observed misbehavior.
**Companion file:** `proposed-edits.patch` (apply from the repo root with `git apply proposed-edits.patch`). Originals are untouched. Items marked **[patched]** are fixed in it; **[your call]** are not.

Overall: the set is unusually disciplined — the "don't invent / use TBD / confirm before overwriting / never self-set Fixed or Done" rules are consistent across commands, and the three plan shapes are deliberately distinct. The problems below are mostly drift between files that grew at different times (the BUG and TASK additions in particular).

## High

**H1 — `/fix-bug` resume path jumps to the wrong steps** · `.claude/commands/fix-bug.md`, step 1 · **[patched]**
Step 1 says to resume an existing bug at "step 5 if root cause is already filled in, otherwise step 4". Step 4 is *assign the next BUG ID and append a new entry*; step 5 is *investigate root cause*; the fix plan is step 6. Followed literally, resuming `BUG-007` with no root cause re-runs step 4 and creates a duplicate entry (`BUG-008`) for the same bug; resuming one with a root cause re-investigates instead of drafting the plan. Likely cause: steps were renumbered when step 3 (requirement search) was added. Fix: skip to 6 / otherwise 5, and say explicitly never to repeat step 4.

**H2 — CLAUDE.md's opening rule contradicts `/plan-task` and `/fix-bug`** · `CLAUDE.md`, intro · **[patched]**
"Every plan traces back to specific requirement IDs. If a request has no requirement backing it, that's a signal to add one." Later sections, `audit-plan`, `plan-task`, and both non-feature templates say bugfix and task plans may legitimately cite zero FR/NFR/UC. Because CLAUDE.md is loaded every session, a model can reasonably stop `/plan-task "bump dependencies"` with "no requirement backing — run /new-requirement?", the opposite of what the command intends. Fix: scope the rule — feature plans cite FR/NFR/UC/IC; bugfix plans cite `BUG-###`; task plans cite `TASK-###`.

**H3 — Example rows cite `FR-001` / `UC-001`, which will be the first *real* IDs** · `docs/bugs/BUG-LOG.md`, `docs/decisions/DECISIONS.md`, `docs/requirements/use-cases.md`, `docs/requirements/functional-requirements.md` · **[patched]**
The example FR row's ID is `_EXAMPLE_`, but the example bug, ADR, and use case reference `FR-001`, and the example FR references `UC-001`. Two failure modes: (a) before real entries exist, `/audit-plan` and `requirements-reviewer` will see dangling references; (b) after the first `/new-requirement` creates a real `FR-001`, any leftover example text silently points at an unrelated requirement — exactly the "an ID that silently means something else" problem the README says the numbering rules exist to prevent. Fix: examples reference `FR-EXAMPLE` / `UC-EXAMPLE` consistently.

## Medium

**M1 — Deprecated / Superseded / Proposed and `_EXAMPLE_` entries are never handled** · `plan-feature`, `audit-plan`, `requirements-reviewer`, `CLAUDE.md` · **[patched]**
CLAUDE.md defines `Deprecated` and `Superseded by` precisely so history stays traceable, but no command says what to do when it meets one. A plan can cite a superseded FR and `/audit-plan` will report "ID exists — OK". Likewise nothing says to ignore placeholder entries while they're still in the docs. Fix: one rule block in CLAUDE.md, plus a matching line in each command that reads the docs; `audit-plan` and the reviewer now flag citations of retired entries.

**M2 — ADRs are listed as "read before planning" but nothing reads them** · `CLAUDE.md`, `plan-feature`, `audit-plan` · **[patched]**
`plan-feature` step 1 reads only `requirements/` and `constraints/`; `audit-plan` doesn't extract or verify `ADR-###` (though `trace` does handle them). A plan can contradict an accepted decision with no warning, and a cited ADR isn't validated. Fix: `plan-feature` reads `DECISIONS.md` and requires an explicit flag when reversing an ADR; `audit-plan` extracts and verifies `ADR-###`.

**M3 — `/deep-dive` never checks for an existing note** · `.claude/commands/deep-dive.md` · **[patched]**
The README and `docs/deep-dives/README.md` justify saving findings with "so the same investigation doesn't happen twice", but the command only checks for a same-slug file at save time (after the work is done). It also says "dated" without a format, so a note can end up with a guessed or relative date, which defeats the staleness check the README describes. Fix: look first, verify rather than redo; use `date +%F` and the short commit hash.

**M4 — Plan-mode advice conflicts with the write step** · `plan-feature` steps 4 & 6, `CLAUDE.md` "Plan mode" · **[patched]**
The command tells the model to prefer plan mode (read-only) and then to write `plans/<slug>-plan.md` in step 6. Within plan mode that write isn't available, so behavior depends on whatever the model improvises. Fix: present the plan, write the file once plan mode is exited/approved.

**M5 — Template placeholders have no fill rules** · `plans/_template-*.md`, `CLAUDE.md` · **[patched]**
Templates use `{{DATE}}` and `**Status:** Draft | In Review | Approved | In Progress | Done`. No command says to set Status to a single value or where the date comes from; a model without a date source will guess one, and a copied option list reads as "all of these". Fix: a short "Filling in templates and logs" section in CLAUDE.md (single starting status, real date via `date +%F`, unknowns stay `TBD`).

**M6 — `TASK-LOG.md` status vocabulary omits `Plan Ready`** · `docs/tasks/TASK-LOG.md` vs `plan-task` step 6 · **[patched]**
The convention line is `Open → In Progress → Done`, but `/plan-task` sets `Plan Ready` and the example entry uses it. `BUG-LOG.md` states "only the user sets Fixed"; the task log's equivalent rule for `Done` lives only in `plan-task`. Fix: add `Plan Ready` to the lifecycle and state the `Done` rule next to it.

**M7 — `/new-requirement` leaves some table columns undefined** · `.claude/commands/new-requirement.md` · **[patched]**
FR and NFR tables have a `Status` column, but the interview never asks for it — so the model must guess (conflicting with "do not guess any of them"). It also doesn't check for an existing overlapping entry, says "highest existing number" without addressing the empty/example case, and the NFR/IC docs say to keep their category lists in sync with no step that does it. Fix: suggest `Proposed` (never default to `Approved`), overlap check, first ID is 001, add new category names to the list.

**M8 — `/audit-plan` doesn't check the plan's own internal consistency** · `.claude/commands/audit-plan.md` · **[patched]**
`_template-plan.md` says the **Requirements traced** list exists "so `/audit-plan` and a human skimming can see coverage at a glance", but the audit never compares that list to the IDs cited in phases, or the **Constraints considered** list to the ICs that apply. Step 5 also says "by cross-referencing `/trace`-style search" — a command can't invoke another command; it should just say to grep. Both fixed.

## Low

**L1 — `/trace` details** · **[patched]** The frontmatter description lists `FR/NFR/UC/IC/ADR` but the body (and CLAUDE.md) also handle `BUG` and `TASK`. `grep -rn` will also search `.git/` and, with `-w` absent, `FR-001` can match inside `FR-0010`. Added `-w --exclude-dir=.git`.

**L2 — `requirements-reviewer` output contract** · **[patched]** Clean files produce no output, which is indistinguishable from "didn't look"; it also would flag `_EXAMPLE_` rows. Added "list every file, write *no findings*" and the placeholder skip. CLAUDE.md never mentions the agent; added one line (delegation is by its `description`, so this is for humans and for discoverability, not required for it to work). **[your call]** Its first two bullets overlap ("untestable language" vs "missing measurable targets on NFRs"); harmless, could be merged.

**L3 — README, Option B** · **[patched]** Telling users to copy `CLAUDE.md` and `.claude/` into an existing repo reads as "overwrite"; `/init-planning` step 3 likewise assumes a blank CLAUDE.md. Added "merge rather than overwrite" to the README. **[your call]** `/init-planning` step 3 could also say "if CLAUDE.md already has content, merge the template sections into it and show the diff first."

**L4 — `arduino-platformio-conventions.md`** · **[patched, partly; verify]**
- Its heading is `##`, the same level as CLAUDE.md's `## Conventions`; pasting it in verbatim produces two `## Conventions` sections. Added a paste-instruction comment.
- `[env:native]` is described as "no `board =`, host compiler" — it also needs `platform = native`. Added.
- It recommends `std::array` as the alternative to heap use on "classic AVR" boards. To my knowledge the classic AVR Arduino toolchain doesn't ship the C++ standard library headers, so `<array>` isn't available there without a third-party library. Softened to "where the toolchain ships it". **Please confirm against your own board before relying on this** — it's from my background knowledge, not tested here.
- It sets C++14 as the baseline; I believe older AVR toolchains default to gnu++11. Reworded to "verify each target toolchain actually builds it". Same caveat as above.
- The file is not mentioned in the README's file tree, and the README claimed "nothing here assumes a particular language or stack". Reworded to call the snippet optional.

## Not patched — your call

- **Duplicate file.** `Claude-outputs/arduino-platformio-conventions.md` is byte-identical to the root copy (both tracked in git). Two copies will drift; the patch edits only the root copy. Keep one — probably the root, given the commit message says it's to be added to CLAUDE.md — and delete or `.gitignore` the other.
- **`BUG-LOG.md` lifecycle starts at `Open`, but `/fix-bug` always creates entries as `Investigating`.** Not wrong (investigation begins immediately), but `Open` is never used by the tooling; either drop it from the convention or let `/fix-bug` use it for logged-but-not-yet-investigated bugs.
- **"Changes nothing" is prose-only for `/trace`, `/audit-plan`, `/deep-dive`.** The only mechanically enforced read-only restriction in the repo is the reviewer subagent's `tools: Read, Grep, Glob`. That's probably fine given the explicit wording, but be aware the guarantee is instruction-level.
- **`.claude/settings.json`** pre-approves Read/Edit only under `docs/**` and `plans/**`. `/init-planning` edits `CLAUDE.md`, so it will prompt — likely intended; noting it in case it isn't.
- **Line wrapping.** A few lines in the patch exceed the files' ~88-column wrap where text was inserted mid-paragraph. Cosmetic.

### Resolution (2026-10-22)

- **Duplicate file:** resolved — `Claude-outputs/arduino-platformio-conventions.md` deleted.
- **`Open` bug status:** kept. `/fix-bug` step 4 now logs entries as `Open` and step 5 moves them to `Investigating`; `BUG-LOG.md` defines `Open`; CLAUDE.md's template-fill rule now names `Draft` (plans) and `Open` (bug/task entries) as the starting values.
- **"Changes nothing" / line wrapping:** accepted as-is.
- **`.claude/settings.json`:** the template is for code repos where Claude reads and edits code, so the allow rules are widened from `docs/**` / `plans/**` to `Read(./**)` and `Edit(./**)` (Edit rules also cover Write). Plan mode still blocks edits. Bash commands (build/test) are not pre-approved — add project-specific rules such as `Bash(make test:*)` per repo.
