# swe-claude

A Claude Code project template for **requirements-driven planning**: write your
functional requirements, non-functional requirements, use cases, and implementation
constraints as plain markdown with stable IDs, and have Claude produce implementation
plans that cite those IDs — instead of planning from a conversation's worth of
half-remembered context.

This is a sibling idea to session-continuity templates like
[claude-config-public](https://github.com/OPENDAP/claude-config-public), which solves
a different problem (picking a research project back up after a month away). This
template solves: *"before Claude plans a feature, has it actually read what we agreed
the system should do, and will the plan say which requirement each step satisfies?"*

## What you get

```
CLAUDE.md                                   # rules Claude reads every session
docs/
  requirements/
    functional-requirements.md              # FR-001, FR-002, ...
    non-functional-requirements.md          # NFR-001, ...
    use-cases.md                             # UC-001, ...
  constraints/
    implementation-constraints.md            # IC-001, ... — hard boundaries
  decisions/
    DECISIONS.md                             # ADR-001, ... — why, not what
  bugs/
    BUG-LOG.md                               # BUG-001, ... — report → root cause → fix
  tasks/
    TASK-LOG.md                              # TASK-001, ... — scoped maintenance/engineering work
  deep-dives/
    README.md                                # convention note; one file per investigation
plans/
  _template-plan.md                          # shape every generated feature plan follows
  _template-bugfix.md                        # shape every generated bugfix plan follows
  _template-task.md                          # shape every generated task plan follows
.claude/
  commands/
    init-planning.md      # /init-planning   — first-time interview & doc seeding
    plan-feature.md        # /plan-feature    — requirements → phased plan
    fix-bug.md              # /fix-bug         — log a bug, find root cause, plan the fix
    deep-dive.md             # /deep-dive       — read-only code investigation, saved findings
    new-requirement.md     # /new-requirement — add an FR/NFR/UC/IC by interview
    plan-task.md             # /plan-task       — log a maintenance/engineering task, draft a plan
    trace.md                # /trace           — find every reference to an ID
    audit-plan.md           # /audit-plan      — check a plan's citations, report only
  agents/
    requirements-reviewer.md  # subagent that reviews docs for vague/untestable language
  settings.json            # example permission scaffold — verify against current docs
```

Every requirement, use case, and constraint gets a stable ID (`FR-001`, `NFR-014`,
`UC-003`, `IC-002`). Plans in `plans/` cite those IDs on every phase. `/trace` and
`/audit-plan` exist so that "does anything actually cover this?" and "did this plan
wander off scope?" are one command away instead of a manual grep.

**Bugs get a parallel ID space, `BUG-###`, deliberately not folded into the FR/NFR/UC
shape.** A bug isn't new capability satisfying a requirement — it's existing behavior
deviating from one (or from something that was never written down in the first
place). `/fix-bug` logs it, investigates root cause read-only, and drafts a bugfix
plan with regression risk and verification steps instead of delivery phases.

**Tasks get a third parallel ID space, `TASK-###`.** Not new capability like an FR,
not a correction like a bug — scoped engineering work such as dependency bumps,
removing dead compile-time directives, or retrofitting tests onto code that predates
them. `/plan-task` logs it and drafts a plan that emphasizes scope (exactly what's
in, what's explicitly out) and how completeness gets checked, since this kind of
task tends to fail quietly — a few instances missed — rather than loudly like a
broken build.

**`/deep-dive` is a standalone, read-only investigation command** for understanding
existing code — onboarding to an unfamiliar module, answering "how does X work,"
or checking whether the code still matches what the requirement docs claim. Findings
get saved to `docs/deep-dives/` so the same investigation doesn't happen twice.
`/plan-feature` and `/fix-bug` both hand off to it when what they need is more than a
quick read-only look.

## Using this as a template

**Option A — GitHub template repository.** Push this to GitHub, then in
**Settings → General**, check **"Template repository."** After that, **Use this
template** on the repo page (or `gh repo create my-project --template
<you>/claude-swe-planning-template`) gives you a fresh copy with its own history for
every new project.

**Option B — copy into an existing repo.** Copy `CLAUDE.md`, `docs/`, `plans/`, and
`.claude/` into a repo you already have (if it already has a `CLAUDE.md` or
`.claude/settings.json`, merge rather than overwrite). Nothing in the core template
assumes a particular language or stack — the `.claude/` folder and `docs/` folder are
additive. Language-specific snippets (currently `arduino-platformio-conventions.md`,
`cpp-conventions.md`, `java-conventions.md`, and `python-conventions.md`) are optional: paste the one you want into the Conventions section of `CLAUDE.md`. Run `/init-planning`
afterward to seed the requirement docs from the existing codebase and an interview,
rather than starting from blank tables.

## First steps in a new project

1. Fill in `{{PROJECT_NAME}}`, `{{ONE_LINE_DESCRIPTION}}`, and `{{CONVENTIONS}}` in
   `CLAUDE.md` — or run `/init-planning` and let the interview do it.
2. Delete the `_EXAMPLE_` rows in each `docs/requirements/*.md` and
   `docs/constraints/*.md` file once you've seen the format — they're there to show
   the table shape, not to be real requirements.
3. Run `/new-requirement` a few times to populate real FRs/NFRs/UCs/ICs, or `/init-
   planning` to do it as one interview.
4. Run `/plan-feature <something>` and check that the plan it writes to `plans/`
   actually cites the IDs you just wrote.

## Design choices worth knowing about

- **IDs are never renumbered or reused.** A dropped requirement is marked
  `Superseded` or `Deprecated`, not deleted — otherwise a plan written last month
  cites an ID that silently means something else now.
- **Claude is instructed not to invent requirements.** If you ask for a feature with
  no backing FR/UC, `/plan-feature` will say so and offer `/new-requirement` rather
  than writing a plausible-sounding one and treating it as fact.
- **`/trace` and `/audit-plan` only report — they never edit.** Traceability checks
  that silently "fix" things by rewriting docs are how requirements quietly drift
  from what was actually agreed.
- **`.claude/settings.json` is an illustrative starting point**, not a verified
  current schema — Claude Code's settings format can change; check
  https://docs.claude.com/en/docs/claude-code/overview before relying on it.

## License

MIT — use it, change it, ship it.
