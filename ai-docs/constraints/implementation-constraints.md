<!-- CONTRACT: things that constrain HOW the system can be built, independent of
     what it must do. These are boundaries, not preferences — if a plan would cross
     one, that's a flag, not a quiet redesign. Delete the _EXAMPLE_ row once you've
     seen the shape. -->

# Implementation Constraints

Convention: `IC-###`. Category is one of: Technology/Platform, Infrastructure/
Deployment, Regulatory/Compliance, Organizational/Team, Third-Party/Integration,
Budget/Timeline, Security, Data integrity (add categories as needed).

| ID | Category | Constraint | Rationale | Impact on design |
|---|---|---|---|---|
| IC-001 | Security | Never send credentials to a probed server — no `Authorization` header, no cookies, no `~/.netrc` | The project's premise is detection without credentials (legacy D11) | Every HTTP session sets `trust_env = False` and `auth = None`; no feature may add credentialed requests to probed servers |
| IC-002 | Security | Never crawl protected content; a single metadata request whose refusal is the datum is allowed | Detecting a server and crawling its data are different problems (legacy D2, D6) | Rules out walking catalogs behind auth; dataset enumeration belongs to `opendap-dataset-catalog` |
| IC-003 | Data integrity | Never fabricate a server, probe result, count or classification; every reported number comes from an actual run whose command is recorded | Registry and status figures must be traceable to measurements (legacy guardrails, D14) | Unmeasured values are reported as "unmeasured", never zero or an estimate; harvesters record candidates and only the probe records truth |
| IC-004 | Security / Third-Party | **Superseded by IC-005.** API keys come from the environment only — never a CLI argument, never logged; outbound HTTP is polite (descriptive User-Agent, rate-limited, honours API paging caps) | A leaked token forced a rotation once; probed servers are hosts someone else pays for | Secret presence is tested with `${VAR:+set}`, never `${VAR:-…}`; all HTTP goes through `src/net.py` sessions |
| IC-005 | Security / Third-Party | Secrets (API keys, tokens, cloud credentials) are never written into any file that can be committed to the repository. They come from the environment or from per-user credential stores outside the repo (e.g. `~/.aws`). Never passed as a CLI argument, never logged; outbound HTTP is polite (descriptive User-Agent, rate-limited, honours API paging caps) | Files in the repo can be pushed to GitHub by accident; a leaked token forced a rotation once; probed servers are hosts someone else pays for | No literal secret in code, config, SQL, fixtures or docs (e.g. DuckDB `CREATE SECRET` uses `PROVIDER credential_chain`, never a literal key); generated files go only to git-ignored paths; secret presence is tested with `${VAR:+set}`, never `${VAR:-…}`. HTTP to *probed servers* still goes through `src/net.py` (IC-001); other HTTP clients (e.g. DuckDB `httpfs` to Common Crawl) may use their own stack but set a descriptive User-Agent |

<!-- A constraint is not the same as a non-functional requirement: an NFR describes
     a quality the system should have (and can trade off); a constraint is a boundary
     condition that plans must work within, full stop. If it's negotiable, it's
     probably an NFR with a priority, not a constraint. -->
