<!-- CONTRACT: quality attributes the system must meet, with a measurable target where
     possible. "Fast" and "user-friendly" are not requirements — a number or a
     testable condition is. Delete the _EXAMPLE_ row once you've seen the shape. -->

# Non-Functional Requirements

Convention: `NFR-###`. Category is one of: Performance, Security, Reliability,
Availability, Scalability, Usability, Maintainability, Compliance, Observability
(add categories as needed — keep the list here in sync). Priority: `Must` / `Should`
/ `Could`. Status as in functional-requirements.md.

| ID | Category | Requirement | Measurable Target | Priority | Status |
|---|---|---|---|---|---|
| NFR-001 | Reliability | The probe's request-rate limit toward probed servers is defined in the repository (code or checked-in config), not left to ad-hoc command-line choice | Limit value 0.5 seconds/request | Should | Approved |
| NFR-002 | Maintainability | The test suite runs entirely offline | `unittest discover -s tests` passes with no network access | Must | Approved |
| NFR-003 | Usability | Every pipeline stage an operator runs (catalog harvest, Censys scan, Common Crawl query, Common Crawl import, probe, export) is documented with its command line, required credentials (named, never their values) and its inputs and outputs | Every runnable entry point in `src/` has a matching section in `README.md` | Should | Proposed |
| NFR-004 | Maintainability | The URL signatures that select candidate OPeNDAP URLs are defined once in code, and every stage that filters by them (e.g. the Common Crawl index query and the Common Crawl import filter) uses that single definition | A unit test shows the query's pattern and the import filter accept and reject the same fixture URLs | Should | Proposed |

<!-- If you genuinely can't state a measurable target yet, write TBD in that column
     rather than a vague phrase — TBD is honest and searchable; "reasonably fast"
     just looks finished. -->
