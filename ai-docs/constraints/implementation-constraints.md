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
| IC-004 | Security / Third-Party | API keys come from the environment only — never a CLI argument, never logged; outbound HTTP is polite (descriptive User-Agent, rate-limited, honours API paging caps) | A leaked token forced a rotation once; probed servers are hosts someone else pays for | Secret presence is tested with `${VAR:+set}`, never `${VAR:-…}`; all HTTP goes through `src/net.py` sessions |

<!-- A constraint is not the same as a non-functional requirement: an NFR describes
     a quality the system should have (and can trade off); a constraint is a boundary
     condition that plans must work within, full stop. If it's negotiable, it's
     probably an NFR with a priority, not a constraint. -->
