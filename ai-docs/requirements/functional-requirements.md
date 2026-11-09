<!-- CONTRACT: what the system must DO. One row per requirement. IDs are permanent —
     never renumbered or reused. Dropped requirements get Status: Deprecated /
     Superseded, not deleted. Delete the _EXAMPLE_ row once you've seen the shape. -->

# Functional Requirements

Convention: `FR-###`, zero-padded, assigned sequentially by next-highest-number.
Priority: `Must` / `Should` / `Could` (MoSCoW). Status: `Proposed` / `Approved` /
`Implemented` / `Deprecated` / `Superseded by FR-0XX`. `Proposed` means drafted but
not yet confirmed. It isn't a proposed *feature*; those live outside this repo until
accepted. `Implemented` is normally set by `/features` when a `FEAT` that cites the
FR reaches `Done`.

| ID | Requirement | Priority | Related Use Cases | Status |
|---|---|---|---|---|
| FR-001 | The system harvests OPeNDAP server URLs from public catalogs (NASA CMR, ERDDAP server lists, ESGF) and normalizes each to a server base URL, stored as one registry row per server | Must | UC-001 | Approved |
| FR-002 | The system discovers OPeNDAP servers that no catalog lists from internet-wide scan data (Censys) and records them as candidates | Must | UC-001 | Approved |
| FR-003 | The system verifies each registered server with an unauthenticated probe and classifies its software (Hyrax / THREDDS / ERDDAP / other) and access status (open / protected / dead / unknown) | Must | UC-001, UC-003 | Approved |
| FR-004 | The system exports the registry as the `servers.json` contract consumed by `opendap-dataset-catalog`, and produces summary reports (`--report`, `--list`) | Must | UC-001, UC-002 | Approved |

<!-- Add new rows via /new-requirement, or by hand — keep the table format. Each
     requirement should be a single testable statement, not a paragraph. If it needs
     a paragraph, it's probably a use case (ai-docs/requirements/use-cases.md) with this
     as one of its acceptance criteria. -->
