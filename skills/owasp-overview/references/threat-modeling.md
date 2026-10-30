# Threat-modeling prompt (STRIDE)

Compact prompt for a new feature or architectural change. Run it at
design time — before code review. Method reference: the OWASP
[Threat Modeling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)
(and, for scoping,
[Attack Surface Analysis](https://cheatsheetseries.owasp.org/cheatsheets/Attack_Surface_Analysis_Cheat_Sheet.html)).
Checked 2026-09-26.

Answer the four questions in order; keep the output to one or two
pages — a threat model nobody reads mitigates nothing.

## 1. What are we building?

- **Assets:** what is worth protecting here (data classes, credentials,
  money-equivalent operations, availability)?
- **Data flows:** how does data move — entry points, stores, external
  services? A plain list or a Mermaid diagram is enough.
- **Trust boundaries:** where does data cross between trust levels
  (internet → app, app → DB, app → third party, tenant → tenant,
  user → admin)? Every boundary crossing is a row in step 2.

## 2. What can go wrong? (STRIDE per boundary crossing)

For each data flow crossing a trust boundary, ask all six:

| STRIDE | Question at this crossing | Typical mitigation class |
|---|---|---|
| **S**poofing | Could the caller be someone else? | Authentication (of users *and* services) |
| **T**ampering | Could the data be modified in flight or at rest? | Integrity: TLS, signatures, MACs |
| **R**epudiation | Could an actor deny having done this? | Audit logging with actor + time |
| **I**nformation disclosure | Could data leak to the wrong party? | Encryption, authorization, minimal responses |
| **D**enial of service | Could this be exhausted or wedged? | Rate limits, quotas, timeouts, fail closed |
| **E**levation of privilege | Could the caller gain rights they lack? | Authorization checks, least privilege |

Record each credible threat as: *actor → action → asset → impact*.

## 3. What are we doing about it?

For every threat from step 2: **mitigate** (name the control and where
it lives), **eliminate** (change the design so the threat is moot),
**transfer** (another system/party owns it — name it), or **accept**
(justify, with an owner). Map each mitigation to the standard it
satisfies ([top10.md](top10.md), [asvs.md](asvs.md),
[api-top10.md](api-top10.md)) so the review and the model stay linked.

## 4. Did we do a good job?

- Every trust boundary from step 1 has a step-2 row; every step-2
  threat has a step-3 decision — no silent gaps.
- Mitigations exist as verifiable requirements (tests, config,
  ASVS IDs), not intentions.
- Revisit the model when the design changes — a threat model is a
  living document, not a launch gate artifact.
