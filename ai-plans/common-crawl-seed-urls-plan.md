# Plan: Common Crawl seed URLs

**Feature:** FEAT-001
**Status:** Draft
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
- NFR-001 (**Proposed**, not yet confirmed; limit value TBD) — governs the request rate
  of any HTTP "crawl up" in Phase 2, and of the probe run that follows each harvest.
- NFR-002 — every new test runs offline from fixtures. DuckDB is never imported by
  the test suite. *(NFR-002 isn't in FEAT-001's `Requirements:` list. It's cited here
  because the plan adds tests. See the plan summary.)*

## Constraints considered

- IC-001 (no credentials to probed servers) — every request to a candidate server
  goes through `net.session()`, which sets `trust_env = False` and `auth = None`.
  This covers the probe and any Phase 2 crawl-up request. No new code path sends
  headers or cookies to a probed host. Respected.
- IC-002 (never crawl protected content) — Phase 1 "crawl up" is pure string
  normalization with zero requests. Phase 2, if it is built, may only request
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
- IC-004 (keys from env only; polite HTTP; **all HTTP goes through `src/net.py`**) —
  Phase 1 makes no network calls (it reads a local file), and Phase 2 uses `net.py`.
  **Phase 3 conflicts with IC-004 and is flagged, not worked around.** DuckDB's
  `httpfs` extension is its own HTTP client, so a query run from this repo bypasses
  `net.py`. The reference script (`cc_dap_discover.py`) also uses `PROVIDER
  credential_chain`, which reads `~/.aws/credentials` and other non-env sources —
  not "environment only". Phase 3 is blocked until Open question 2 is decided.

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
the number of admitted candidates (NFR-001 is still TBD). The `.html` URLs that
crawlers do see (e.g. `.../erddap/griddap/x.html`) normalize correctly, so recall
from marker-bearing URLs should be good.

### Phase 2 — HTTP crawl-up for URLs without a marker (conditional)

**Goal:** For seed URLs with no service marker, find the actual service root by
requesting successive *parent* paths. Build this only if Phase 1's measurements
show it's needed.
**Satisfies:** FR-002, FR-001, FR-003, NFR-001 (Proposed)

Steps:
1. Measure after Phase 1. Count the `commoncrawl` servers that sit at a host root,
   came from a sample URL with a DAP suffix, and that the probe left `unknown` or
   non-DAP even after the D28 subpath fallback. If that count is negligible, mark
   this phase not needed in the work log and stop.
2. Otherwise add a bounded walk. For each such seed, request the parent
   directories of its path, longest first, through `net.session()`, with a hard
   cap of N requests per host (N TBD). Stop at the first DAP signature, which
   becomes the base URL, or at any 401/403/login redirect, which is recorded and
   not followed. Never request children or follow links (IC-002).
3. The rate limit and per-host cap are checked into code, not left to the command
   line (NFR-001).
4. Offline tests drive the walk against canned responses.

**Risks:** This is the one place the feature generates new traffic to third-party
hosts. It is blocked on Open question 1 and depends on NFR-001's limit, which is
still TBD.

### Phase 3 — Run the Common Crawl index query from this repo (blocked)

**Goal:** Replace the out-of-band step with an in-repo `--query CRAWL` mode that
runs the DuckDB query from `cc_dap_discover.py` and feeds Phase 1 directly.
**Satisfies:** FR-002, UC-001

Steps (depend on Open question 2):
1. Add `duckdb` as an optional dependency in `requirements.txt`, imported lazily
   inside the query function so the test suite never needs it (NFR-002).
2. Move the query's URL regex into the module, shared with Phase 1's filter so the
   two can't drift.
3. Configure access as decided under Open question 2. Whatever the decision, never
   use `PROVIDER credential_chain`. Set a descriptive `custom_user_agent`.
4. Offline tests cover only SQL construction. The query itself is checked by a
   recorded manual run (IC-003).

**Risks:** Each crawl's index is tens of GB of Parquet scanned client-side, which
takes a long time and is slow on a laptop. DuckDB is a large binary dependency in a
project with a minimal-dependencies convention. A 2.0.0 cp311 wheel exists for this
machine's `.venv`.

### Phase 4 — Requirement wording and operator docs

**Goal:** Make the written requirements say what the code now does.
**Satisfies:** FR-002, UC-001

Steps:
1. Depending on Open question 5, run `/new-requirement` (or a supersede) so FR-002
   and UC-001 step 3 name web-crawl sources, not only Censys.
2. Document the two-step operator flow (query, then import) where the Censys flow
   is documented.

## Open questions

1. **What does "crawl up" mean in issue #1?** Is it Phase 1's offline truncation
   at the service-root marker, or HTTP requests up parent paths? This blocks
   Phase 2 only. Phase 1's measurement should inform the answer.
2. **How should Phase 3 meet IC-004?** Options: (a) don't do Phase 3, and keep the
   query out of the repo with Phase 1 as the integration point, which avoids the
   conflict entirely; (b) read the index anonymously over
   `https://data.commoncrawl.org/` with no AWS credentials, getting the file list
   through `net.py`, with DuckDB still doing the Parquet reads — this needs an ADR
   recording an explicit IC-004 exception for DuckDB's HTTP client; (c) S3 with
   keys from env vars only, which needs the same exception. This blocks Phase 3.
3. **Which crawls, and how many?** One crawl to start, as in the reference script,
   or several for recall? It doesn't block Phase 1, which takes any file and a
   `--crawl` tag.
4. **What default cap on new candidates?** TBD until Phase 1 step 8's measured dry
   run. It doesn't block writing Phase 1, only its default value.
5. **Should FR-002 be broadened** to cover web-crawl data explicitly, or is
   "internet-wide scan data" meant broadly? This blocks Phase 4 and affects
   traceability, not code.
6. **What is NFR-001's rate limit?** Still TBD and `Proposed`. It blocks Phase 2's
   rate setting and matters for probing a large Phase 1 batch.

## Out of scope

- Crawling *down* from any seed URL, or enumerating datasets. That belongs to
  `opendap-dataset-catalog` (IC-002).
- Using Common Crawl WARC *content* (page bodies) or a Common Crawl hit as
  evidence of software or access. Only the probe classifies (IC-003).
- Changes to the probe's classification logic, or to the `servers.json` contract.
  `source_tags` already carries any new source name.
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
