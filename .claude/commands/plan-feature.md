---
description: Draft an implementation plan for a feature, citing the FR/NFR/UC/IC IDs it satisfies
argument-hint: <FEAT-### or feature name / short description>
---

Draft an implementation plan for: $ARGUMENTS

If `$ARGUMENTS` is empty, ask for a `FEAT-###` or a short feature name or description
and stop.

## Steps

1. **Resolve the feature entry.** Read `ai-docs/features/FEATURE-LOG.md`.
   - If `$ARGUMENTS` is a `FEAT-###` in the log, use that entry. If its status is
     `Done`, `Won't Do`, or `Superseded`, say so and ask before planning it.
   - Otherwise, look for an entry that matches the description. If one does,
     confirm it with the user and use it.
   - If none matches, say so and offer to run `/new-feature` first, the same way
     step 3 offers `/new-requirement`. Don't plan an unlogged feature unless the
     user explicitly says to; if they do, the plan's `Feature:` line is `None`.
   - If the entry already has a `Plan:` path, treat this as a revision of that plan
     (see step 7), not a new one.

2. **Read every file in `ai-docs/requirements/` and `ai-docs/constraints/` in full**, right
   now — don't rely on what they said earlier in this session. They may have changed.
   Also read `ai-docs/decisions/DECISIONS.md`: a plan that reverses an accepted ADR must
   say so explicitly. Skip `_EXAMPLE_` / `*-EXAMPLE` placeholder entries.

3. **Identify which existing `FR-###` / `NFR-###` / `UC-###` entries this feature
   is actually backed by.** Don't use `Deprecated` entries; follow `Superseded by` to the
   replacement; flag `Proposed` entries as unconfirmed. If none apply — the feature isn't traceable to anything
   written down — stop and say so plainly. Offer to run `/new-requirement` first.
   Do not invent a plausible-sounding requirement and plan against it; a plan built on
   an invented requirement is indistinguishable from a real one until someone checks.
   Compare what you find with the `FEAT` entry's `Requirements:` list. If they
   differ, say how, and ask whether to update the entry. Don't change it silently.

4. **Identify every `IC-###` that constrains this design space.** Read them, not just
   their IDs — a constraint you skim past is one you'll violate by accident.

5. **Think through the approach before writing anything.** Prefer plan mode for this —
   research the existing code relevant to the feature (read-only) so the plan reflects
   what's actually there, not an assumption about it. If the relevant area is
   unfamiliar or large enough to need real mapping before you can plan against it, run
   `/deep-dive` on it first and come back with findings rather than researching inline
   under time pressure to produce a plan.

6. **Draft the plan using the shape in `ai-plans/_template-plan.md`.** Concretely:
   - The `**Feature:**` line names the `FEAT-###` from step 1.
   - Every phase cites the FR/NFR/UC IDs it satisfies.
   - The "Constraints considered" section cites every relevant IC and states how the
     plan respects it.
   - If the plan would violate a constraint, **say so explicitly in the plan** — a
     flagged tension the user can decide on, not a silent workaround and not a silent
     violation.
   - Open questions are genuine unknowns, not gaps papered over with a guess.
   - The `## Work log` section gets its first entry: a `Plan created` heading stamped
     with the output of `date '+%F %H:%M %Z'` (run it — never guess), the initiating
     prompt quoted verbatim (`/plan-feature $ARGUMENTS`, plus any other instructions
     the user gave in that message), and a one-to-three-line summary of what was done.

7. **Write the plan to `ai-plans/<slug>-plan.md`**, where `<slug>` is the feature name in
   kebab-case. (If you're in plan mode, present the plan and write the file once plan
   mode is exited or approved — don't try to write it from inside plan mode.) If that file already exists, ask before overwriting — don't clobber a
   plan that might be mid-review. If the user wants the existing plan revised
   instead, keep its work log and append a new timestamped entry summarizing the
   revision and quoting the prompt that asked for it.

   Once the file is written, set the `FEAT` entry's `Plan:` field to its path. Don't
   change the feature's `Status`; that's done with `/features`.

8. **Summarize back to the user:**
   - The `FEAT-###` the plan is for, and that its `Plan:` field now points here.
   - Which FR/NFR/UC IDs the plan covers.
   - Any relevant requirements you found that the plan does *not* address, and why.
   - Any flagged constraint conflicts.
   - Any open questions that block a phase.

Do not start implementing. This command produces a plan file for review, nothing else.
