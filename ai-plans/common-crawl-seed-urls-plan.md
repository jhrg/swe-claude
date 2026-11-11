# Plan: Common Crawl seed URLs

**Feature:** FEAT-001
**Status:** Approved
**Created:** 2026-11-09

## Summary

Add Common Crawl's URL index as a candidate source alongside Censys and crt.sh.
OPeNDAP-looking URLs from a crawl become seed URLs. Each is "crawled up" to its server
root and admitted as a `commoncrawl` candidate, and the existing probe decides which
ones are real. Common Crawl URLs carry full dataset paths, unlike Censys's bare
`host:port`. That gives the normalizer a real service root and the probe's content
stage a real dataset to test for gating.

## Requirements traced

- FR-002 — the core of the feature. It finds servers no catalog lists and records them
  as candidates. **Wording gap:** FR-002 says "internet-wide *scan* data (Censys)", and
  Common Crawl is web-*crawl* data. This plan reads FR-002 broadly. See Open question 5.
- FR-001 — each seed URL is normalized to a server base URL, one registry row per
  server, through the existing `normalize.to_base_url`.
- FR-003 — the new candidates are verified and classified by the existing
  unauthenticated probe. The harvester never sets `software` or `auth_status`.
- UC-001 — step 3 ("System adds scan-derived candidates") gains a second source.
  UC-001 names only Censys today. See Open question 5.
- NFR-001 (Approved, Should; 0.5 seconds per request) — sets the request rate for
  Phase 2's HTTP crawl-up, and applies to the probe run that follows each harvest.
  The probe currently sets no pause (`net.get` defaults to `pause=0.0`). See Out of
  scope.
- NFR-002 — every new test runs offline from fixtures. DuckDB is never imported by
  the test suite.
- NFR-003 (**Proposed**, Should) — every operator-run stage is documented in
  `README.md` with its command, the credentials it needs (named only) and its
  inputs and outputs. Phase 4 documents the new stages.
- NFR-004 (**Proposed**, Should) — candidate-URL signatures are defined once and
  shared by every filter. Phase 3 shares the query's pattern with Phase 1's import
  filter and adds a test that they agree.

## Constraints considered

- IC-001 (no credentials to probed servers) — every request to a candidate server
  goes through `net.session()`, which sets `trust_env = False` and `auth = None`.
  This covers the probe and any Phase 2 crawl-up request. No new code path sends
  headers or cookies to a probed host. Respected.
- IC-002 (never crawl protected content) — Phase 1 "crawl up" is pure string
  normalization with zero requests. Phase 2's HTTP crawl-up may only request
  *ancestors* of a URL already in hand. It never follows links or listings into
  descendants, and it stops at the first DAP signature or any 401/403/login
  redirect. The probe's content stage already rewrites data URLs (`.dods`, `.ascii`,
  ...) to a metadata request (`.dmr`/`.dds`) before asking, so a Common Crawl
  `sample_url` pointing at data never causes a data download. Respected.
- IC-003 (never fabricate; counts come from recorded runs) — Common Crawl hits are
  candidates only (source `commoncrawl`). The harvester prints its own command line,
  input file and crawl ID with its counts, so every number is tied to a run. A
  `--dry-run` reports counts without writing. Default caps are left TBD until a
  measured dry run, not estimated. Respected.
- IC-005 (no secrets in any committable file; no secrets as CLI arguments or in logs;
  polite HTTP; HTTP to probed servers goes through `net.py`, and other clients may
  use their own stack with a descriptive User-Agent). It supersedes IC-004, whose
  "all HTTP through `net.py`" and "env only" rules would have blocked Phase 3.
  - Phase 1 makes no network calls; it reads a local file.
  - Phase 2's crawl-up goes to probed hosts, so it uses `net.py`.
  - Phase 3's DuckDB `httpfs` traffic goes to Common Crawl, not to a probed server,
    so its own HTTP client is allowed. It sets `custom_user_agent`. It reads S3
    credentials with `PROVIDER credential_chain`, which may use `~/.aws`. No key,
    token or `CREATE SECRET` literal appears in code, SQL, fixtures or docs.
  - Generated files go only under the git-ignored `data/cache/`.
  - The "descriptive User-Agent" part depends on **BUG-001** (`Fix Planned`):
    `net.CONTACT`, which is embedded in every User-Agent, is a placeholder URL
    that doesn't exist. Phase 2 inherits it through `net.session()`. Phase 3 builds
    its `custom_user_agent` from the same `net.CONTACT`, not a second copy.

  Respected once BUG-001 is fixed. Until then, Phase 2's traffic carries a broken
  contact URL. See the Phase 2 risks.

No accepted ADRs exist in `ai-docs/decisions/DECISIONS.md` (only the example), so
none is contradicted. Legacy D6, D14, D25 and D28 are honoured: harvesters record
candidates and the probe records truth, and bare hosts get the D28 subpath fallback.

## Phases

### Phase 1 — Import a Common Crawl URL list as candidates (offline)

**Goal:** Run the Common Crawl query outside this repo (with the issue's
`cc_dap_discover.py`, or Athena). Import its `candidate_urls.txt` and admit one
`commoncrawl` candidate per normalized server root, then probe.
**Satisfies:** FR-002, FR-001, FR-003, UC-001, NFR-002

Steps:
1. Add `src/harvest_commoncrawl.py`, shaped like `fingerprint.run_censys`. Gather
   candidates, split them into known and new against `servers.base_url`, cap the new
   ones, tag known servers first, then admit new ones and optionally run
   `probe.run`. `--source censys` already shows the pattern.
2. Input is a text file with one URL per line. For a CSV whose first column is
   `url` (the reference script's `.csv`), read only that column. Its rows currently
   have one more field than its header (tld is written but not declared), so read
   the URL by position, not by header.
3. Filter each URL with a Common Crawl-specific test: `normalize.looks_like_dap`,
   plus the reference script's DAP signatures such as `/dap/` and `.dsr`. Then
   reduce it with `normalize.to_base_url`. For this source, "crawl up" means
   truncating the path at the service-root marker. That already exists and makes
   no requests.
4. Group by base URL. For each server, keep the URL that
   `probe.content_probe_urls` accepts with the longest path as `sample_url`, so the
   probe's content stage can test gating. Record `source_ref` as the crawl ID
   (e.g. `CC-MAIN-2026-21`), supplied by a required `--crawl` flag because the
   input file doesn't carry it.
5. CLI: `--db`, `--input`, `--crawl`, `--max-candidates` (default TBD after the
   first dry run), `--dry-run`, `--no-probe`, `--probe-workers`. Print the argv,
   the input path and the counts (IC-003).
6. Add `commoncrawl` to the `sources.source` comment in `schema.sql`. No schema
   change is needed.
7. Tests in `tests/test_harvest_commoncrawl.py` with a small fixture under
   `tests/fixtures/`, all offline (NFR-002). Cover the filter, normalization and
   grouping, `sample_url` choice, the cap order, idempotent re-import, and that no
   `software`/`auth_status` is written.
8. Run a measured `--dry-run` on a real crawl's output, record the command and
   counts, then set the `--max-candidates` default from that.

**Risks:** Common Crawl is noisy. `/dap/` and bare `.das`/`.dds` URLs with no
service marker collapse to the host root, and that can mean thousands of
non-OPeNDAP hosts. The cap and the probe limit the cost, but probe load grows with
the number of admitted candidates. The probe doesn't yet pace itself to NFR-001's
0.5 s/request (see Out of scope). The `.html` URLs that
crawlers do see (e.g. `.../erddap/griddap/x.html`) normalize correctly, so recall
from marker-bearing URLs should be good.

### Phase 2 — HTTP crawl-up to find the service root

**Goal:** The user confirmed on 2026-11-09 that issue #1's "crawl up" means HTTP
requests. For seed URLs with no service marker, which Phase 1 can only reduce to
the host root, find the real service root by requesting successive *parent*
paths of the seed URL.
**Satisfies:** FR-002, FR-001, FR-003, NFR-001

Steps:
1. Choose which seeds to walk: Common Crawl URLs with no `SERVICE_MARKERS`
   segment, such as `/dap/` mounts or bare `.dds`/`.das` files. Seeds with a
   marker already reach their service root offline in Phase 1.
2. For each one, request the parent directories of its path, deepest first,
   through `net.session()` (IC-001). Use a hard cap of N requests per host (N
   TBD), and walk each host once even if it has several seeds. Reuse the probe's
   classification signatures to recognise a DAP root, rather than a second copy.
3. Stop at the first DAP signature, which becomes the candidate's base URL. Also
   stop at any 401/403 or login redirect: record it and don't follow it. Never
   request anything below a seed or follow links, listings or catalogs (IC-002).
   If the walk finds nothing, fall back to Phase 1's host-root candidate, so the
   probe still sees it and nothing is dropped silently.
4. If the base URL found has no service marker, the registry needs to accept it.
   The upsert takes any base URL, but a deep root like `https://h/pub/dap/` won't
   match `probe._paths_for`'s marker-based ordering. Check that the probe's
   generic `PROBE_PATHS` work against such a root, and record the result.
5. Define the pacing as a module constant: 0.5 s between requests, passed as
   `net.get(..., pause=...)` (NFR-001). Define the per-host cap the same way. Neither
   comes from the command line. Each host's walk is sequential, so at minimum
   requests to any one host are 0.5 s apart. Whether the limit is also global
   across concurrent workers is Open question 6. Expose `--no-crawl-up` to skip
   the phase.
6. Offline tests drive the walk against canned responses (NFR-002). Cover:
   ancestor order, the per-host cap, stopping on 401/403/redirect, the host-root
   fallback, and that no request goes below the seed.

**Risks:** This is the one place the feature generates new traffic to third-party
hosts. The volume is roughly the number of marker-less seed hosts times N (N is
still TBD). At 0.5 s/request, run time also scales with that volume divided by the
number of workers. Its User-Agent carries `net.CONTACT`, a placeholder URL
(BUG-001). Fix BUG-001 before running Phase 2 against real hosts, so operators can
reach us.

### Phase 3 — Run the Common Crawl index query from this repo

**Goal:** Replace the out-of-band step with an in-repo `--query CRAWL` mode that
runs the DuckDB query from `cc_dap_discover.py` and feeds Phase 1 directly.
**Satisfies:** FR-002, UC-001, NFR-004 (Proposed)

Steps (IC-005):
1. Add `duckdb` as an optional dependency in `requirements.txt`, imported lazily
   inside the query function so the test suite never needs it (NFR-002).
2. Move the query's URL regex into the module, shared with Phase 1's filter so the
   two can't drift. Add a test that both accept and reject the same fixture URLs
   (NFR-004, Proposed).
3. Read the S3 index with DuckDB's `PROVIDER credential_chain`, which may use
   `~/.aws`, as in the reference script. No key ID, secret or token appears in any
   repo file, CLI argument or log line. Set `custom_user_agent` to a string built
   from `net.CONTACT`, the same contact the `net.py` sessions use, so a BUG-001
   fix covers both (IC-005).
4. Write intermediate output, such as a saved candidate list, only under
   `data/cache/`, which is already git-ignored. Never write it to the repo root,
   as the reference script does.
5. Offline tests cover only SQL construction. The query itself is checked by a
   recorded manual run (IC-003).

**Risks:** Each crawl's index is tens of GB of Parquet scanned client-side, which
takes a long time and is slow on a laptop. DuckDB is a large binary dependency in a
project with a minimal-dependencies convention. A 2.0.0 cp311 wheel exists for this
machine's `.venv`.

### Phase 4 — Requirement wording and operator docs

**Goal:** Make the written requirements say what the code now does.
**Satisfies:** FR-002, UC-001, NFR-003 (Proposed)

Steps:
1. Depending on Open question 5, run `/new-requirement` (or a supersede) so FR-002
   and UC-001 step 3 name web-crawl sources, not only Censys.
2. Bring the rest of UC-001 in line:
   - **Preconditions:** add AWS credentials readable by `credential_chain` (env or
     `~/.aws`) for Phase 3's in-repo query, alongside `CENSYS_PAT`.
   - **Alternate flows:** add the case where no AWS credentials are found, so the
     query stage fails and the operator can still import a file (Phase 1).
   - **Related requirements:** cite IC-005 in place of the superseded IC-004.
3. Document the two-step operator flow (query, then import) where the Censys flow
   is documented. Give the query, import and crawl-up stages each a `README.md`
   section with the command line, the credentials it needs (named only) and its
   inputs and outputs (NFR-003, Proposed).

## Open questions

1. ~~**What does "crawl up" mean in issue #1?**~~ **Resolved 2026-11-09:** it means
   HTTP requests up parent paths, so Phase 2 is committed work.
2. ~~**IC-004 revision.**~~ **Resolved 2026-11-09:** IC-005 supersedes IC-004, so
   Phase 3 is unblocked.
3. **Which crawls, and how many?** One crawl to start, as in the reference script,
   or several for recall? It doesn't block Phase 1, which takes any file and a
   `--crawl` tag.
4. **What default cap on new candidates?** TBD until Phase 1 step 8's measured dry
   run. It doesn't block writing Phase 1, only its default value.
5. **Should FR-002 be broadened** to cover web-crawl data explicitly, or is
   "internet-wide scan data" meant broadly? This blocks Phase 4 and affects
   traceability, not code.
6. **What is NFR-001's 0.5 seconds/request measured against?** Per worker or per
   host, which the current plan meets by pacing each sequential walk? Or global
   across all concurrent workers, which would need a shared limiter? The second is
   about eight times slower with the probe's default eight workers. Phase 2 can be
   built per host now. If the answer is global, the limiter changes.
7. **Per-host crawl-up cap N.** TBD. It doesn't block writing Phase 2. Set it from
   the depth of marker-less seed paths seen in Phase 1's dry run.

## Out of scope

- Crawling *down* from any seed URL, or enumerating datasets. That belongs to
  `opendap-dataset-catalog` (IC-002).
- Using Common Crawl WARC *content* (page bodies) or a Common Crawl hit as
  evidence of software or access. Only the probe classifies (IC-003).
- Changes to the probe's classification logic, or to the `servers.json` contract.
  `source_tags` already carries any new source name.
- Making the existing probe meet NFR-001. It currently calls `net.get` with no
  pause. That's a deviation in code this feature doesn't own, and it should be
  logged separately with `/fix-bug`. Until it's fixed, the probe run after a large
  Common Crawl import goes faster than NFR-001 allows.
- Athena or any paid-query setup.
- Shodan, or new Censys queries.

## Work log

### 2026-11-09 19:36 MST — Plan created

**Prompt:**

> /plan-feature FEAT-001

Re-read the requirement, constraint and ADR docs. Read `src/fingerprint.py`
(Censys/crt.sh drivers), `normalize.py`, `probe.py`, `net.py`, `schema.sql` and the
issue's `cc_dap_discover.py`. Drafted four phases: Phase 1 is ready, Phase 2 is
conditional on a measurement, Phase 3 is blocked on an IC-004 decision, and Phase 4
covers requirement wording. Nothing is implemented.

### 2026-11-09 23:25 MST — Revised: crawl-up confirmed, IC-004 restated

**Prompt:**

> Phase 2 does mean crawl up
> Phase 3 Not all HTTP access has to go through net.py and we can read credentials from ~/.aws. The real constriant is on files in the repo because thaey can be checked into GitHub by acident.

Phase 2 is now committed, with no measurement gate. It defines which seeds to walk,
falls back to the host root when the walk finds nothing, and checks that the probe
handles a root with no service marker. Phase 3 now uses `credential_chain` with no
secrets in repo files and output under `data/cache/`. It's blocked only on revising
IC-004's text, which hasn't been done. Open question 1 is resolved and Open
question 2 is restated.

### 2026-11-09 23:30 MST — Revised against IC-005 and the now-Approved NFR-001

**Prompt:**

> /plan-feature FEAT-001

Re-read the requirement, constraint and ADR docs. IC-004 is superseded by IC-005,
so Phase 3 is unblocked and the constraint section is rewritten around IC-005.
NFR-001 is now Approved at 0.5 s/request. Phase 2 sets that pacing as a module
constant and gets a new Open question 6 on whether the limit is per host or
global. Open question 2 is resolved and Open question 7 (crawl-up cap N) is added.
The probe not pacing itself to NFR-001 is flagged as out of scope, for `/fix-bug`.

### 2026-11-10 17:38 MST — Fixed audit findings 1–3

**Prompt:**

> fix findings 1-3, then run /new-requirement for  "documenting the operator flow" and /new-requirement for  Phase 3's "sharing the URL regex with Phase 1's filter"

From `/audit-plan`:
1. Removed the out-of-date note that NFR-002 wasn't in FEAT-001's list.
2. Added BUG-001 (placeholder `net.CONTACT` in the User-Agent) as a dependency under
   IC-005 and as a Phase 2 risk. Phase 3's `custom_user_agent` now reuses
   `net.CONTACT`.
3. Expanded Phase 4 to update UC-001's preconditions (AWS credentials), its
   alternate flows (no credentials found), and its IC-004 → IC-005 citation.

The two `/new-requirement` runs follow. Their IDs get cited in the plan once they
exist.

### 2026-11-10 17:41 MST — Cited NFR-003 and NFR-004

Added NFR-003 (operator docs) and NFR-004 (one shared definition of the URL
signatures), both Proposed and Should, through `/new-requirement`. They're now
cited in Requirements traced, in Phase 3 (NFR-004, with a test that the query and
the import filter agree) and in Phase 4 (NFR-003, README sections). That closes
audit finding 5's two uncited steps. `--no-crawl-up` is still uncited.
