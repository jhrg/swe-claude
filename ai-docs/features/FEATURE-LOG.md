<!-- CONTRACT: one entry per ACCEPTED feature — a unit of new capability that groups
     the FR/NFR/UC IDs it delivers and the plan that builds it. Proposed features
     (ideas not yet accepted) do NOT live here; they live wherever this project
     collects ideas (see Conventions in CLAUDE.md), and an entry's Origin field points
     back to one. A FEAT is not a requirement: it never states behavior itself, it
     cites the FR/NFR/UC rows that do. Delete the FEAT-EXAMPLE section once you've
     seen the shape. -->

# Feature Log

Convention: `FEAT-###`, sequential, never renumbered. Status: `Planned` →
`In Progress` → `Done` (or `Won't Do` / `Superseded by FEAT-0XX`). Change status
with `/features FEAT-### <status>`, not by hand, so the Status history line gets
written and, on `Done`, the cited FRs are set to `Implemented`. **Only the user sets
`Done`.** It's a claim about verified, shipped work, not something to infer from a
plan's phases being finished.

Whether a plan exists is shown by the `Plan:` field (`None yet` or a path), not by
the status. A feature's status is its lifecycle; a plan's own `Status` describes only
the plan document.

---

## FEAT-EXAMPLE — Password reset by email

*(delete this section once you've seen the shape)*

**Status:** In Progress
**Created:** YYYY-MM-DD
**Requirements:** FR-EXAMPLE, UC-EXAMPLE
**Constraints:** None found
**Plan:** `ai-plans/password-reset-plan.md`
**Origin:** https://github.com/OWNER/REPO/issues/42

**Summary:** Users who forget their password can reset it through a time-limited
link sent to their registered email address.

**Status history:**
- YYYY-MM-DD — logged as `Planned`
- YYYY-MM-DD — `Planned` → `In Progress`

---

## FEAT-001 — Common Crawl seed URLs

**Status:** Planned
**Created:** 2026-11-09
**Requirements:** FR-001, FR-002, FR-003, UC-001, NFR-001, NFR-002
**Constraints:** IC-001, IC-002, IC-003, IC-005
**Plan:** `ai-plans/common-crawl-seed-urls-plan.md`
**Origin:** https://github.com/OPENDAP/opendap-registry/issues/1

**Summary:** Use the Common Crawl URL index, queried with DuckDB, as a second source
of seed URLs alongside Censys, then crawl up from each URL to find its server root
and record it as a candidate.

**Status history:**
- 2026-11-09 — logged as `Planned`

---

<!-- Add new features via /new-feature, and change status via /features — keep the
     section format: metadata lines, then Summary, then Status history (append-only,
     oldest first). Requirements must list real FR/NFR/UC IDs; a feature with none
     isn't ready to log — add them with /new-requirement first. -->
