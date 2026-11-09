# Bugfix Plan: User-Agent contact is a placeholder URL that doesn't exist

**Bug:** BUG-001
**Status:** Draft
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

1. **Decide the address (blocking, TBD).** You chose an email address, but the exact
   address hasn't been given yet. Also undecided is whether it replaces the URL or sits
   next to it, for example `(+https://github.com/OPENDAP/opendap-registry;
   mailto:<address>)`. Whatever goes here is sent to every server and is public in the
   source, so a role or list address will get less spam than a personal one.
2. **Change the single source.** Set `CONTACT` in `src/net.py` to the chosen value. The
   five `USER_AGENT` f-strings stay as they are, because they already read
   `net.CONTACT`. The `+` prefix is the usual convention for a URL; if `CONTACT`
   becomes a bare email, change the f-strings to a `mailto:` form or drop the `+` so
   the result is still well formed.
3. **Add a regression test** (offline, per NFR-002). It asserts that `net.CONTACT`
   doesn't contain `your-org` and matches the agreed form (an email, or URL plus
   email), and that each module's `USER_AGENT` contains `net.CONTACT`.

This is the narrow fix, and that's enough here: the defect is one constant, and it
already has a single point of definition. Making the contact configurable through an
environment variable or config file isn't needed. Nothing calls for it, and it would
open the door to a run that silently has no contact.

## Regression risk

Low. `CONTACT` is only used in User-Agent strings. No test fixture matches on the User-
Agent (`tests/` has no reference to `USER_AGENT` or `CONTACT`), and no server-side
behavior we depend on keys off it. The one real risk is a badly formed User-Agent if
the email goes in with the `+` URL prefix still in front of it. Step 2 handles that.

## Verification

1. Re-run the reproduction from BUG-001:
   `.venv/bin/python -c "from src import probe; print(probe.USER_AGENT)"`. The output
   now shows the agreed contact and no `your-org`.
2. Do the same for `harvest_cmr`, `harvest_erddap`, `harvest_esgf` and `fingerprint`.
3. If a URL is kept, open it and confirm it resolves for an outside user. A private
   repo returns a 404 to them.
4. `.venv/bin/python -m unittest discover -s tests` passes offline, including the new
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
