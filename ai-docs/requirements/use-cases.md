<!-- CONTRACT: actor-driven flows through the system. Each use case is what ties
     functional requirements together into something a real user actually does.
     Delete the _EXAMPLE_ use case once you've seen the shape. -->

# Use Cases

Convention: `UC-###`. Each use case follows the template below. Link related `FR-###`
/ `NFR-###` IDs at the bottom so `/trace` can find them in both directions.

---

## UC-001 — Build or refresh the registry

**Actor(s):** Registry operator (OPeNDAP developer)

**Trigger:** Operator wants a new or updated registry — first build, or a periodic
refresh

**Preconditions:** `.venv` set up; for the scan stage, `CENSYS_PAT` in the environment
(run under `zsh -i`)

**Main flow:**
1. Operator initializes the DB (`--init`) or reuses the existing one
2. System harvests server URLs from CMR, ERDDAP lists and ESGF and upserts one row per
   normalized server base URL
3. System adds scan-derived candidates (Censys)
4. System probes every server without credentials and records software and access
   status
5. Operator runs the report to see the result

**Alternate / exception flows:**
- `CENSYS_PAT` absent → scan stage has no credential; operator re-runs under `zsh -i`
  (catalog stages and probe need no keys)
- A catalog or server is unreachable → harvest/probe records what it got; nothing is
  fabricated (IC-003)

**Postconditions:** Registry rows updated in place, never duplicated; `first_seen`
unchanged, `last_checked` advanced

**Related requirements:** FR-001, FR-002, FR-003, FR-004, NFR-001, IC-001, IC-002,
IC-003, IC-004

---

## UC-002 — Hand the server list to `opendap-dataset-catalog`

**Actor(s):** Registry operator; consumer is the `opendap-dataset-catalog` project

**Trigger:** The dataset catalog needs a current list of servers

**Preconditions:** Registry has been built (UC-001)

**Main flow:**
1. Operator runs `--export-servers <path>`
2. System writes `servers.json` in the agreed contract shape
3. `opendap-dataset-catalog` reads the file as its input

**Alternate / exception flows:**
- TBD

**Postconditions:** `servers.json` reflects the registry as of the export

**Related requirements:** FR-004

---

## UC-003 — Re-probe servers to update classifications

**Actor(s):** Registry operator

**Trigger:** Classifications are stale, or some servers are `unknown`

**Preconditions:** Registry contains servers

**Main flow:**
1. Operator runs the probe, optionally with `--only-unknown`
2. System re-probes the selected servers without credentials
3. System updates software and access status for each

**Alternate / exception flows:**
- TLS handshake fails on this machine's old TLS stack → server stays `unknown`, not
  `dead` (legacy D22)

**Postconditions:** Classifications and `last_checked` updated

**Related requirements:** FR-003, NFR-001, IC-001, IC-002

---

<!-- Copy the template above for each new use case. Sequential UC-### heading,
     same subsection order, so /trace can parse it. -->
