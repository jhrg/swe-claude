<!-- CONTRACT: deviations from intended behavior, tracked from report through root
     cause to fix. Not a duplicate of ai-docs/requirements/ — a bug means something
     already documented (or implicitly assumed) isn't holding true in practice.
     Delete the BUG-EXAMPLE section once you've seen the shape. -->

# Bug Log

Convention: `BUG-###`, sequential, never renumbered. Status: `Open` → `Investigating`
→ `Fix Planned` → `Fixed` → (or `Won't Fix` / `Duplicate of BUG-0XX`). `Open` means
logged with reproduction steps but root cause not yet looked at (`/fix-bug` sets it
at step 4 and moves to `Investigating` at step 5; an entry logged by hand starts here
too). **Only the user sets a bug to `Fixed`** — that's a claim about verified
deployed behavior, not something to self-report from having written a fix plan.

---

## BUG-001 — User-Agent contact is a placeholder URL that doesn't exist

**Severity:** Medium
**Status:** Fix Planned
**Reported:** 2026-11-08
**Related requirements:** IC-004 (polite HTTP: descriptive User-Agent). No FR/NFR/UC
documents the contact-in-User-Agent behavior.
**Fix plan:** `ai-plans/bugfix-placeholder-contact-url-plan.md`

**Reproduction steps:**
1. From the repo root, run
   `.venv/bin/python -c "from src import probe; print(probe.USER_AGENT)"`
2. Observe `opendap-registry-probe/0.1 (+https://github.com/your-org/opendap-registry)`
3. Open `https://github.com/your-org/opendap-registry` and get a 404 (no such repo)

The same placeholder appears in the User-Agent of `harvest_cmr`, `harvest_erddap`,
`harvest_esgf` and `fingerprint`, so it goes to every probed server, catalog API and
Censys.

**Expected behavior:** The User-Agent carries a working contact, a project email
address (exact address TBD), so a server operator can identify the traffic and reach
us (IC-004).

**Actual behavior:** The User-Agent names a placeholder GitHub URL that does not
resolve.

**Root cause:** `src/net.py:17` defines
`CONTACT = "https://github.com/your-org/opendap-registry"`. It is scaffold text, there
since the first implementation commit (`108e788`, 2026-07-24) and never replaced. Every
module builds its `USER_AGENT` from `net.CONTACT` (`probe.py:57`, `harvest_cmr.py:39`,
`harvest_erddap.py:39`, `harvest_esgf.py:46`, `fingerprint.py:49`). No test checks the
User-Agent, so nothing flagged the placeholder.

---

<!-- Add new bugs via /fix-bug, or by hand — keep the section format: metadata lines,
     then Reproduction / Expected / Actual / Root Cause. A bug with no reproduction
     steps yet is a symptom report, not a bug entry — get concrete steps before
     assigning an ID if at all possible. -->
