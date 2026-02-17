# Protocol U v1.0.2 — Formalization & Orphan-Consensus Protocol (Clean Draft)
**Status:** Normative  
**Revision:** v1.0.2 (2026-02-13) — INSUFFICIENT→INSUFFICIENT; orphan consensus binding added; quorum treated as witness attestations; gap codes aligned to Appendix B.
**Role:** Produce computable *logical* artifacts (scope, terms, assumptions, falsifiers, operationalization) for SVP; never issues TRUE_SYN/FALSE_SYN.  
**Downstream:** STEP 4 orphan routing (if NOVEL_ORPHAN), Protocol S (simulation evidence), STEP 5 (evidence builder), STEP 6 (SVP-EVAL)

---

## U0) Non-Goals and Guardrails

### U0.1 Truth/Zone prohibition
Protocol U MUST NOT:
- assign `TRUE_SYN / FALSE_SYN / UNKNOWN_SYN`,
- assign zones (TRUTH/LIE/HYPOTHESIS).

Protocol U MAY assign only *logical* statuses:
`logical_status ∈ {PROVED, DISPROVED, CONSISTENT, INSUFFICIENT, BLOCKED_*}`

### U0.2 Output discipline
Protocol U outputs **U-artifacts** only. Every claim leaving U MUST be bound to:
- explicit scope,
- explicit term definitions (TermTable),
- explicit assumptions,
- explicit falsifiers,
- (if READY_FOR_SIM) explicit operationalization mapping.

---

## U0.3 Orphan-consensus binding (Policy-bound)

When STEP 4 outputs `class = NOVEL_ORPHAN`, the system MAY invoke Protocol U in **Orphan-Consensus mode** if:
- `policy.orphan_consensus.enabled=true`

Policy MUST provide:
- `policy.orphan_consensus.protocol_ref` (expected: `PROTOCOL_U_v1_0`)
- `policy.orphan_consensus.quorum.min_humans`
- `policy.orphan_consensus.quorum.min_agents`
- `policy.orphan_consensus.required_fields[]`

**Human role:** humans act only as *witnesses of artifacts* (attestations, term definitions, scope notes), not judges.
Quorum counts *attestations present*, not normative approvals.

Failure modes (fail-closed):
- missing required orphan fields → `logical_status=BLOCKED_CONTEXT_MISSING` + EvidenceGap `"provenance_missing_fields"` if identity is unbound.
- missing attestations to satisfy quorum → EvidenceGap `"witness_attestation_missing"`.

---

## U1) Inputs

- `CO` (Claim Object candidate)
- domain/horizon declaration (may be incomplete)
- optional evidence refs (E_*), but U does not “weigh truth,” only uses them as definitions/constraints.

---

## U2) Parsing & Claim Typing

### U2.1 Claim typing
Assign:
`claim_type ∈ {DESCRIPTIVE, CAUSAL, DESIGN, NORMATIVE, META}`

### U2.2 Decomposition
If claim contains multiple independent sub-claims:
- MUST split into `{CO_i}` with separate scopes and falsifiers,
- else set `BLOCKED_SPLIT_REQUIRED`.

---

## U3) Scope Pack (U-SCOPE-*) — Mandatory

### U3.1 Required fields (MUST)
`U-SCOPE-*` MUST include:
- `scope_id`
- `domain`
- `horizon_class ∈ {short, medium, long}`
- `risk_class ∈ {LOW, MED, HIGH}` (MUST for DESIGN/NORMATIVE)
- `boundaries.conditions[]` (what must hold)
- `boundaries.exclusions[]` (what is explicitly out of scope)
- `measurement.context` (where/when/for whom the claim applies)
- `success_definition` (in operational terms; may reference U-OPMAP)
- `stop_rule` (when evaluation stops as INSUFFICIENT)

### U3.2 Compute-readiness gate
If scope is missing or ambiguous:
- MUST emit `BLOCKED_NEEDS_SCOPE`.

---

## U4) TermTable Pack (U-TERMS-*) — Mandatory

### U4.1 Required fields (MUST)
`U-TERMS-*` MUST include:
- `terms[]: {term, definition, units?, allowed_range?, aliases?, notes?}`
- `undefined_terms[]` (if any remain)

If undefined terms remain that are essential:
- MUST emit `BLOCKED_NEEDS_DEFS`.

---

## U5) Assumptions Pack (U-ASSUMPTIONS-*) — Mandatory

### U5.1 Required fields (MUST)
`U-ASSUMPTIONS-*` MUST include:
- `assumptions[]: {assumption_id, statement, necessity, falsifiable?, notes?}`
- `assumption_dependencies[]` (links to terms/scope)
- `assumption_risk_notes` (especially for HIGH)

---

## U6) Derivation Pack (U-DERIVATION-*) — Optional but recommended

### U6.1 Allowed outputs
- proof sketch / derivation trace
- counterexample outline
- consistency argument

### U6.2 Required fields if produced
- `derivation_id`
- `method ∈ {formal_proof, counterexample, consistency_check, reduction}`
- `steps[]` (referencing terms/scope)
- `dependencies[]` (assumptions used)
- `failure_points[]` (what would break the derivation)

---

## U7) Falsifier Pack (U-FALSIFIERS-*) — Mandatory

### U7.1 Required fields (MUST)
`U-FALSIFIERS-*` MUST include:
- `observable_predictions[]: {pred_id, statement, depends_on[]}`
- `tests[]` each with:
  - `test_id`
  - `metric_id`
  - `description`
  - `direction ∈ {maximize, minimize, within_band}`
  - `threshold: number | {low, high}`
  - `window: {method: tail_fraction|fixed, tail_fraction?, steps?}`
  - `severity ∈ {info, warn, fail}`
- `failure_modes[]: {failure_mode_id, trigger_ref, notes}`

---

## U7.1) Operationalization Mapping Pack (U-OPMAP-*) — NEW, Mandatory for READY_FOR_SIM

### U7.1.1 Purpose
Make U output **machine-ingestible** for Protocol S and Step 6: claim → variables → metrics → pass criteria.

### U7.1.2 Required fields (MUST)
`U-OPMAP-*` MUST include:
- `claim_id`
- `horizon_class`
- `risk_class`
- `scope_ref: U-SCOPE-*`
- `variables[]: {var_id, definition_ref, units?, allowed_range?}`
- `metrics[]` each with:
  - `metric_id`
  - `description`
  - `direction ∈ {maximize, minimize, within_band}`
  - `band? {low, high}`
  - `aggregation ∈ {mean, p95, max, min, rate}`
  - `window {method, tail_fraction?, steps?}`
  - `normalization {scale, clip}`
- `pass_criteria[]` each with:
  - `criterion_id`
  - `metric_id`
  - `predicate ∈ {>=, <=, within_band, no_violations}`
  - `threshold: number | {low, high}`
  - `severity ∈ {info, warn, fail}`
- `failure_modes[]` (MUST reference U-FALSIFIERS failure_mode_id)
- `auditability`:
  - `assumptions_refs[]`
  - `falsifiers_refs[]`
  - `derivation_refs[]`

---

## U8) Complexity & Routing

### U8.1 Complexity score
Compute `U_complexity ∈ [0..100]` using:
- number of terms,
- number of coupled variables,
- presence of cross-element causality,
- horizon length,
- risk class.

### U8.2 Routing rules (NORMATIVE)
- If missing scope → `BLOCKED_NEEDS_SCOPE`
- If missing definitions → `BLOCKED_NEEDS_DEFS`
- If multi-claim and not split → `BLOCKED_SPLIT_REQUIRED`
- Else if `U_complexity > 70` and claim_type ∈ {CAUSAL, DESIGN, NORMATIVE}:
  - MUST require split OR narrow scope; otherwise `BLOCKED_SPLIT_REQUIRED`
- `READY_FOR_SIM` is permitted **only if**:
  - `U-SCOPE-*` present with exclusions + measurement,
  - `U-FALSIFIERS-*` present,
  - `U-OPMAP-*` present,
  - and split/narrow-scope conditions satisfied.
- Otherwise:
  - `READY_FOR_EMPIRICS` (if data is required),
  - or `INSUFFICIENT` (if logically consistent but not decidable).

---


---

## U8) Orphan-Consensus Mode (NOVEL_ORPHAN) (Normative)

### U8.1 Required input fields (from STEP 4)
Protocol U in orphan mode MUST receive an `OrphanIncidentRecord` (OIR) or equivalent fields:
- `co_id`
- `risk_class ∈ {LOW, MED, HIGH}`
- `suspected_domain.tags[]` (may be empty)
- `routing.reason` (string)
- `kgr_ref` (pointer)

These correspond to `policy.orphan_consensus.required_fields[]`.
If any field missing → `logical_status=BLOCKED_CONTEXT_MISSING` and emit EvidenceGap `"missing_hash_bindings"` or `"provenance_missing_fields"` as applicable.

### U8.2 Quorum as attestations (No-judging)
Minimum attestations required:
- humans: `N_h ≥ policy.orphan_consensus.quorum.min_humans`
- agents: `N_a ≥ policy.orphan_consensus.quorum.min_agents`

Each attestation MUST be a signed artifact reference with:
- `witness_id`
- `artifact_ref` (TermTable/ScopeNote/Counterexample/Operationalization draft)
- `artifact_hash` (sha256)
- `timestamp_utc`

If quorum not met → `logical_status=INSUFFICIENT` and emit EvidenceGap `"witness_attestation_missing"`.

### U8.3 Orphan deliverables
Orphan mode MUST output:
- `U-REPORT` (structured): scope, terms, assumptions, candidate falsifiers
- optional `U-OPMAP` if operationalization is possible without simulation
- `u_orphan_routing` recommendation (non-normative): which domain bucket to attach

If scope/terms cannot be produced → emit EvidenceGaps:
- `"missing_scope"`, `"missing_terms"`.

### U8.4 Output contract
Protocol U MUST NOT emit TRUE/FALSE zones. It emits only:
- `logical_status` + artifact set
- `evidence_gaps[]` (Appendix B codes)
- `next_actions[]` (e.g., request more terms, request more witnesses, request simulation)

## U9) Outputs

Protocol U MUST emit:
- `U-SCOPE-*`
- `U-TERMS-*`
- `U-ASSUMPTIONS-*`
- `U-FALSIFIERS-*`
- `U-OPMAP-*` (iff READY_FOR_SIM)
- optionally `U-DERIVATION-*`

And a summary status object:
- `U-STATUS: {logical_status, blockers[], next_step, artifact_refs[]}`

—

# U-REPORT-* (U-STATUS object) — template
u_report_template:
  artifact_id: "U-REPORT-0001"
  class: "U_REPORT"
  protocol: "U"
  version: "1.0.1"
  co_id: "CO_..."
  time_window:
    observed_from_utc: "YYYY-MM-DDThh:mm:ssZ"
    observed_to_utc: "YYYY-MM-DDThh:mm:ssZ"

  logical_status: "INSUFFICIENT"   # PROVED|DISPROVED|CONSISTENT|INSUFFICIENT|BLOCKED_*
  blockers: []                     # list of BLOCKED_* codes
  next_step: "S"                   # S|EMPIRICS|SPLIT|STOP
  artifact_refs:
    scope: "U-SCOPE-..."
    terms: "U-TERMS-..."
    assumptions: "U-ASSUMPTIONS-..."
    falsifiers: "U-FALSIFIERS-..."
    opmap: "U-OPMAP-..."           # required if READY_FOR_SIM
    derivation: []                 # optional list

  complexity:
    U_complexity: 0
    notes: ""

  notes:
    - "U does not assign TRUE_SYN/FALSE_SYN"


—

# U-OPMAP-* — template (operationalization mapping)
u_opmap_template:
  artifact_id: "U-OPMAP-0001"
  class: "U_OPMAP"
  protocol: "U"
  version: "1.0.1"
  co_id: "CO_..."

  scope_ref: "U-SCOPE-..."
  horizon_class: "medium"   # short|medium|long
  risk_class: "MED"         # LOW|MED|HIGH
  flow_affecting: true

  variables:
    - var_id: "v1"
      definition_ref: "TermTable.term_name"
      units: null
      allowed_range: null

  metrics:
    - metric_id: "m1"
      description: "..."
      direction: "maximize"           # maximize|minimize|within_band
      band: null                      # {low,high} if within_band
      aggregation: "mean"             # mean|p95|max|min|rate
      window: { method: "tail_fraction", tail_fraction: 0.6 }
      normalization: { scale: 0.10, clip: true }

  pass_criteria:
    - criterion_id: "pc1"
      metric_id: "m1"
      predicate: ">="
      threshold: 0.2
      severity: "fail"                # info|warn|fail

  failure_modes:
    - failure_mode_id: "FM1"
      trigger: "pc1"
      notes: "..."

  auditability:
    assumptions_refs: ["U-ASSUMPTIONS-..."]
    falsifiers_refs: ["U-FALSIFIERS-..."]
    derivation_refs: []
