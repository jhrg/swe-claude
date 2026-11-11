# Bugfix Plan: User-Agent contact is a placeholder URL that doesn't exist

**Bug:** BUG-001
**Status:** Approved
**Created:** 2026-11-08

## Summary

Every outbound request names `https://github.com/your-org/opendap-registry` as its
contact. That URL is a scaffold placeholder that doesn't exist. This fix replaces it
with a real project contact email and adds a test so that a placeholder can't slip
back in.

## Root cause

`src/net.py:17` sets `CONTACT = "https://github.com/your-org/opendap-registry"`. It has
been there unchanged since the first implementation commit (`108e788`, 2026-07-24).
Five modules build their `USER_AGENT` from it:

- `src/probe.py:57`: `opendap-registry-probe/0.1 (+{net.CONTACT})`
- `src/harvest_cmr.py:39`, `src/harvest_erddap.py:39`, `src/harvest_esgf.py:46`,
  `src/fingerprint.py:49`: `opendap-registry/0.1 (+{net.CONTACT})`

So the placeholder goes to every probed server, to the CMR/ESGF/ERDDAP catalogs, and
to Censys. No test looks at the User-Agent, which is why nobody noticed.

## Related requirements

- IC-004: polite outbound HTTP with a descriptive User-Agent. A User-Agent whose only
  contact is dead doesn't let an operator reach us, so it isn't descriptive in the way
  that matters.
- No FR/NFR/UC states that the User-Agent must carry a working contact. The rule
  exists only as the IC and the legacy guardrails (`TASKS.md`, `PLAN.md`). The user
  chose not to add a requirement for it now.

## Fix approach

1. **The contact (decided 2026-11-10).** Email only: `registry@opendap.org`, a role
   address, since the value goes to every server and is public in the source. An
   earlier choice of repo URL plus email was dropped because
   `https://github.com/OPENDAP/opendap-registry` returns 404 to anyone not logged in.
2. **Change the single source.** Set `CONTACT = "mailto:registry@opendap.org"` in
   `src/net.py`. The five `USER_AGENT` f-strings already read `net.CONTACT`; drop the
   `+` from each one, since that prefix is the convention for a web URL. The result
   is, for example, `opendap-registry-probe/0.1 (mailto:registry@opendap.org)`.
3. **Add a regression test** (offline, per NFR-002). It asserts that `net.CONTACT`
   doesn't contain `your-org` and equals `mailto:registry@opendap.org`, and that each
   of the five modules' `USER_AGENT` contains `(mailto:registry@opendap.org)` with no
   `+` before it.

This is the narrow fix, and that's enough here: the defect is one constant, and it
already has a single point of definition. Making the contact configurable through an
environment variable or config file isn't needed. Nothing calls for it, and it would
open the door to a run that silently has no contact.

## Regression risk

Low. `CONTACT` is only used in User-Agent strings. No test fixture matches on the User-
Agent (`tests/` has no reference to `USER_AGENT` or `CONTACT`), and no server-side
behavior we depend on keys off it. The one real risk is a badly formed User-Agent if
a `+` is left in front of the `mailto:`. Step 2 removes it, and step 3 checks for it.

## Verification

1. Re-run the reproduction from BUG-001:
   `.venv/bin/python -c "from src import probe; print(probe.USER_AGENT)"`. The output
   now shows `opendap-registry-probe/0.1 (mailto:registry@opendap.org)` and no
   `your-org`.
2. Do the same for `harvest_cmr`, `harvest_erddap`, `harvest_esgf` and `fingerprint`.
3. `.venv/bin/python -m unittest discover -s tests` passes offline, including the new
   test.

## Out of scope

- The probe's rate limit (NFR-001, Proposed, value TBD). It's also covered by IC-004
  but is a separate piece of work.
- The `/0.1` version string in each User-Agent, and making it consistent across
  modules.
- Making the contact configurable (see Fix approach).

## Work log

### 2026-11-08 18:57 MST — Plan created

**Prompt:**

> /fix-bug CONTACT in src/net.py:17 is still the placeholder
> https://github.com/your-org/opendap-registry. That means the probe's User-Agent sends
> servers a dead contact link, which falls short of the politeness rule in IC-004.

Logged BUG-001 (Medium, cites IC-004 only). Traced the root cause to scaffold text in
`src/net.py:17`, which reaches all five User-Agents. Drafted a narrow fix plus a
regression test. Blocked on the exact contact email address.

### 2026-11-10 17:44 MST — Resumed; contact form decided

**Prompt:**

> /fix-bug BUG-001

Picked up BUG-001 at `Fix Planned`; root cause and plan were already in place.
Created branch `jhrg/placeholder-contact-url` off `main`. The user picked the
contact form, repo URL plus email, and the address is still TBD.
`https://github.com/OPENDAP/opendap-registry` returned 404 when fetched without
login, which is now noted under Fix approach step 1. Still blocked on (a) the email
address and (b) whether the repo goes public or the URL is dropped.

### 2026-11-10 17:52 MST — Contact address given

**Prompt:**

> The contact email is `registry@opendap.org`

Recorded `registry@opendap.org` as the contact address in Fix approach step 1, so the
email blocker is cleared. One blocker remains:
`https://github.com/OPENDAP/opendap-registry` returns 404 without login. Either the
repo goes public before this ships, or `CONTACT` falls back to
`mailto:registry@opendap.org` alone.

### 2026-11-10 17:53 MST — Contact form changed to email only; plan unblocked

**Prompt:**

> Use the email alone

Dropped the repo URL (it 404s without login). Rewrote Fix approach step 1 as decided:
`CONTACT = "mailto:registry@opendap.org"`, with the `+` removed from the five
User-Agent f-strings. Tightened the regression test and the verification steps to
match, and removed the "open the URL" check. Nothing blocks the plan now; it's ready
to implement.
