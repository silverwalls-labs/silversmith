---
name: threat-modeling
description: Structured per-feature threat modeling with STRIDE over data flows. Use when designing a new feature, data flow, or architecture change, or reviewing one for security — whenever "what could go wrong here?" needs a structured answer.
---

# threat-modeling

Run a structured threat model when a feature is designed or reviewed:
decide whether a model is warranted, enumerate threats with STRIDE over a
data flow diagram, and turn the result into a tracked threat register
that lives next to the code. The scope is the per-feature modeling
process — general secure-coding rules (input validation, crypto choices,
dependency hygiene) are the project's global security reference, not this
skill; here they only appear as mitigations assigned to specific threats.

## Rules

Normative. Each MUST/NEVER doubles as a review check: to review a new
feature or an existing model, walk this list top to bottom and flag
every violation, even ones you were not asked about.

- **MUST threat-model new features, new data flows, and architecture
  changes — NOT every PR.** A one-line bugfix doesn't need a model; a
  new service that handles user data does. Model when a trigger fires:
  new external boundary, new data class (PII, secrets, payment), new
  trust boundary, new third-party integration, or new privileged role.
- **MUST keep the effort proportional:** a 30-minute model for a small
  feature, a full session for a new service. NEVER let the ceremony be
  the reason no model exists.
- **MUST revisit the model when the architecture changes.** A stale
  model is worse than none — it gives false confidence. Close threats
  when the feature is removed or the boundary is eliminated.
- **MUST draw the data flow diagram first** — components, data stores,
  external entities, and trust boundaries (network, process, privilege,
  org). The DFD is the canvas: threats are enumerated per flow and per
  component, not brainstormed in the abstract.
- **MUST apply STRIDE per element:** Spoofing, Tampering, Repudiation,
  Information disclosure, Denial of service, Elevation of privilege —
  walked against each component, store, and boundary-crossing flow.
- **Every threat MUST name the asset at risk and the trust boundary
  being crossed.** A threat without an asset is a non-threat: drop it.
- **NEVER stop at "an attacker could X".** Each threat states its
  precondition (what the attacker needs) and its impact (what is lost).
- **MUST rank every threat** — High/Med/Low by default; upgrade to
  likelihood × impact for larger features where H/M/L stops
  discriminating.
- **MUST assign a mitigation to every non-accepted threat:** prevent,
  detect, respond, or accept. NEVER accept a risk without a reason, an
  owner, and an expiry — an accepted risk with no owner and no expiry is
  just an untracked risk.
- **MUST produce a tracked threat list** (issues or a register), each
  entry with asset, threat, ranking, mitigation, owner, status. A model
  that lives in a slide deck and is never updated is theater.
- **MUST link each threat to the control that mitigates it and to the
  code/config that implements the control**, so the register stays
  verifiable against the codebase.
- **MUST store the model next to the code** (e.g.
  `docs/threat-models/<feature>.md`) so it is discoverable and
  versioned with the architecture it describes.

## Modeling flow

1. **Check the triggers.** No new boundary, data class, integration, or
   privileged role → no model; say so and stop. Otherwise size the
   effort to the change.
2. **Draw the DFD** — components, data stores, external entities, trust
   boundaries — starting from
   [references/template.md](references/template.md).
3. **Enumerate STRIDE** per element and per boundary-crossing flow;
   for each threat record asset, boundary, precondition, and impact.
4. **Rank and mitigate:** H/M/L per threat; assign
   prevent/detect/respond mitigations with owners, or accept with
   reason + owner + expiry.
5. **Record the register** at `docs/threat-models/<feature>.md` (or as
   tracked issues), linking each threat to its control and
   implementation.
6. **Revisit on architecture change;** close threats whose feature or
   boundary is gone. A worked end-to-end pass:
   [references/example-file-upload.md](references/example-file-upload.md).

## Notes and edge cases

- **Tooling stays lightweight:** OWASP Threat Dragon, ThreatModeler, or
  plain markdown + a diagram (Mermaid/Excalidraw) are all valid — the
  tool that gets used is the right one. Don't let tooling become the
  barrier: a 20-minute whiteboard session captured as a photo + a
  bullet list is a valid model.
- **Templates beat blank pages:** start from the one-page STRIDE
  template in [references/template.md](references/template.md) rather
  than an empty file.
- **Repudiation is the letter teams skip.** If nothing in the feature
  writes an audit trail, say so explicitly in the register — either as
  a threat with a logging mitigation or as an accepted risk.
- **Third-party components still get modeled:** you don't control their
  internals, but the flow into and out of them crosses a trust boundary
  you do control — model that crossing.
- **Existing features with no model:** don't retrofit models across the
  backlog. Model on the next architecture-changing touch, when the DFD
  has to be drawn anyway.
