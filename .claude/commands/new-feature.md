---
description: Log an accepted feature as a FEAT-### entry, citing the FR/NFR/UC IDs that back it — doesn't plan it
argument-hint: <short feature name or description>
---

Log a new feature: $ARGUMENTS

This records a feature the project has *accepted*. Proposed features (ideas not yet
accepted) aren't this repo's concern. They live wherever the project collects ideas
(see Conventions in `CLAUDE.md`), and the entry's `Origin` field points back to one.

## Steps

1. **Get a description.** If `$ARGUMENTS` is empty, ask for a short feature name or
   description and stop.

2. **Check it isn't already logged.** Read `ai-docs/features/FEATURE-LOG.md`. If an
   existing entry already covers this feature (any status, including `Won't Do`),
   show it and ask whether to proceed with a new entry or work with that one, using
   `/features` to change its status or `/plan-feature FEAT-###` to plan it.

3. **Read every file in `ai-docs/requirements/` in full**, right now — don't rely on
   what they said earlier in this session. Also read
   `ai-docs/constraints/implementation-constraints.md`. Skip `_EXAMPLE_` /
   `*-EXAMPLE` placeholder entries.

4. **Identify the FR/NFR/UC IDs that back this feature.** Propose a list and confirm
   it with the user. Don't cite `Deprecated` entries; follow `Superseded by` to the
   replacement; flag `Proposed` entries as not yet confirmed. **If nothing backs the
   feature, stop**: say so plainly and offer `/new-requirement`. Log nothing. A
   feature entry built on an invented requirement looks just like a real one until
   someone checks. Also note any `IC-###` that obviously constrains it; finding none
   is normal.

5. **Interview for the remaining fields — don't guess any of them:**
   - Title (short)
   - Summary: one or two sentences on what the feature delivers
   - Origin: where the proposal came from (an issue URL, a tracker link, a meeting
     note). If the user doesn't know or there's no record, write `TBD`.

6. **Assign the next `FEAT-###` ID.** Find the highest existing number in
   `FEATURE-LOG.md` (ignore `FEAT-EXAMPLE`; with no real entries yet, the first ID
   is `FEAT-001`). Never reuse a number, including one marked `Won't Do` or
   `Superseded`.

7. **Show the drafted entry, then append it on confirmation**, following the
   existing section format: `Status: Planned`, `Created:` today's date (run
   `date +%F` — never guess), `Plan: None yet`, and a first Status history line
   `<date> — logged as Planned`. Don't touch other entries.

8. **Report back:** the `FEAT-###` ID, the requirements it cites (flagging any
   `Proposed` ones), and the next useful commands: `/plan-feature FEAT-###` to plan
   it, `/features` to see the board.

Do not draft a plan, and do not change any FR/NFR/UC status. This command only logs
the feature.
