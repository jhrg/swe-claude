---
description: Show the feature board (FEAT-### by status), or change a feature's status — on Done, marks its FRs Implemented
argument-hint: (optional) FEAT-### <Planned | In Progress | Done | Won't Do | Superseded by FEAT-###>
---

Features: $ARGUMENTS

Read `ai-docs/features/FEATURE-LOG.md` in full first. Skip `FEAT-EXAMPLE`.

## With no arguments: show the board (read-only)

Print the features grouped by status, in this order: `In Progress`, `Planned`,
`Done`, `Won't Do`, `Superseded`. Omit empty groups. For each entry show the ID,
title, requirement IDs, plan (or `None yet`), and Origin. After the board, list
anything that looks stale, without changing it:

- a `Planned` or `In Progress` feature with `Plan: None yet`;
- a cited requirement ID that no longer exists, or is now `Deprecated` or
  `Superseded`;
- a `Plan:` path that doesn't exist.

If there are no real entries, say so and suggest `/new-feature`. Change nothing.

## With `FEAT-### <status>`: change a status

1. **Validate.** The ID must exist in the log. The status must be one of `Planned`,
   `In Progress`, `Done`, `Won't Do`, or `Superseded by FEAT-###`, where the target
   exists and isn't the same feature. Anything else: say what's wrong and stop.
   Setting the status a feature already has is a no-op; say so.

2. **Only on the user's word.** Make a status change only when the user asked for
   that change in this message. Never infer `Done` from a plan's phases being
   finished. It's the user's claim about verified, shipped work. If this command
   was run with no explicit status, show the board instead.

3. **Update the entry.** Change its `Status:` line and append a line to its
   `Status history`: `<date> — <old> → <new>`, using today's date from
   `date +%F` (never guess it). Don't touch other entries or other fields.

4. **On `Done`, mark the cited FRs `Implemented`** in
   `ai-docs/requirements/functional-requirements.md`. For each `FR-###` in the
   entry's `Requirements:` field:
   - Already `Implemented`: leave it.
   - `Deprecated` or `Superseded`: leave it and mention it in the report.
   - Also cited by another `FEAT` whose status isn't `Done`, `Won't Do`, or
     `Superseded`: **ask before changing it**. Name the other feature, because that
     feature may still need the FR.
   - Otherwise: set its `Status` cell to `Implemented`.

   Change only the `Status` cell of those rows. Leave NFR and UC statuses alone.
   Report each FR you changed, and each you skipped and why.

5. **GitHub, optional.** If the entry's `Origin` is a GitHub issue URL
   (`https://github.com/<owner>/<repo>/issues/<n>`) and `gh auth status` succeeds,
   *offer* to post a comment on the issue with the new status and, for `Done`,
   `Won't Do`, or `Superseded`, to close it. Ask each time; this is visible to other
   people. Run `gh issue comment` / `gh issue close` only after the user agrees. If
   `gh` is missing or not logged in, or `Origin` isn't a GitHub issue, skip this step
   without comment.

6. **Report back:** the old and new status, the FR changes from step 4, and any
   GitHub action taken.
