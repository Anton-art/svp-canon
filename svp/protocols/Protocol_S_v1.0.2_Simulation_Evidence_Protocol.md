# Protocol S v1.0.2 — Simulation Evidence Protocol (Clean Draft)
**Status:** Normative  
**Revision:** v1.0.2 (2026-02-13) — aligned robustness wording to INSUFFICIENT; added stress_pass_rate and borrowed_flag; precondition gate emits gaps; Step 6 compatibility.
**Role:** Produce simulation evidence `E_sim` (S-REPORT) and optional consensus evidence `E_cons`; never issues TRUE_SYN/FALSE_SYN.  
**Depends on:** U-OPMAP / U-FALSIFIERS / U-SCOPE  
**Downstream:** SVP-EVAL consumes E_sim under No-Guessing and coupling gates.

---

## S0) Non-Goals and Guardrails

### S0.1 Truth/Zone prohibition
Protocol S MUST NOT:
- assign `TRUE_SYN / FALSE_SYN / UNKNOWN_SYN`,
- assign zones.

Protocol S outputs only:
- `E_sim` as `S-REPORT-*`,
- optional `E_cons` as `S-CONSENSUS-*`.

---

## S1) Inputs

- Claim `CO` + `U-artifacts`:
  - `U-SCOPE-*`
  - `U-FALSIFIERS-*`
  - `U-ASSUMPTIONS-*`
  - `U-OPMAP-*` (mandatory if simulation proceeds)
- `S_config`:
  - `profile_id/profile_version` (e.g., `SCS_PROFILE_DEFAULT_v0.1`)
  - `horizon_class ∈ {short, medium, long}`
  - `risk_class ∈ {LOW, MED, HIGH}`
  - `thresholds: {min_cycles, min_stress_pass, eps_ab_delta}`
  - `normalization_params: {alpha, tau, K_scale, E_scale}` OR profile reference
  - `independence_requirements`
- Coupling graph:
  - `graph_version` (+ recommended `graph_hash`)

---

## S2) Preconditions Gate (BLOCKED_NEEDS_U)

Simulation MUST NOT run unless:
- scope is complete (`U-SCOPE-*`),
- falsifiers exist (`U-FALSIFIERS-*`),
- operationalization exists (`U-OPMAP-*`),
- assumptions are explicit (`U-ASSUMPTIONS-*`).

If any missing → emit `S-REPORT` with `robustness.verdict=INSUFFICIENT`, and set:
- `gap_codes += {missing_scope, missing_opmap, missing_terms}` as applicable,
- `quality.limitations += missing_inputs`.


---

## S3) Roles and Mirror Protocol

### S3.1 Roles
- Builder (I_ops): constructs baseline + intervention mapping
- Critic (I_gov): constructs adversarial variants and audits assumptions
- Witness (human): attests artifacts, scope, hashes (integrity-only)

### S3.2 Mirror protocol (A/B)
Both Builder and Critic MUST run:
  - baseline model
  - intervention model
under identical declared normalization params and profile bindings.

If InterventionSet differs materially between A and B → `INSUFFICIENT` and record `intervention_mapping_disagreement`.

---

## S4) Scoreboard (from U-OPMAP)

### S4.1 Metrics
All metrics, windows, and pass criteria MUST come from `U-OPMAP-*` / `U-FALSIFIERS-*`.
No ad-hoc metrics are allowed.

### S4.2 Outputs to compute
From nominal runs:
- `element_signals` and mapped `element_findings` (per profile mapping)
- optional `edge_signals` if enabled/required

---

## S4.1) Stress Suite (mandatory for flow-affecting and HIGH)

### S4.1.1 Required coverage
S MUST form `stress_suite.cases` with:
- node coverage (HUM/AI/BIO/TEC/SOC),
- critical edge coverage:
  - inbound to AI and SOC,
  - outbound from HUM and AI,
  - plus closure on any intervened edges,
- mixed shocks if `flow_affecting=true`.

### S4.1.2 PASS/FAIL computation
A stress case PASS/FAIL MUST be computed only from `pass_criteria` in U-OPMAP/U-FALSIFIERS.

`stress_suite.pass_rate` MUST be:
- fraction of cases that pass,
- inconclusive cases counted as fail (conservative).

---

## S5) Mirror-Safe Robustness (A/B + stress)

### S5.1 Compute ab_delta
Compute `builder_vs_critic_delta (ab_delta)` as a normalized distance between:
- vector of `S_e` across 5 elements,
- plus (optional) edge signal vector over `critical_edges_used`.

Compare with `eps_ab_delta` from profile.

### S5.2 Robustness verdict (Normative)
- If coverage missing (nodes or critical edges) → `robustness.verdict=INSUFFICIENT` and `gap_codes += stress_coverage_incomplete`
- Else if `ab_delta > eps_ab_delta` → `robustness.verdict=INSUFFICIENT` and `gap_codes += ab_mirror_missing`
- Else if `stress_suite.pass_rate < thresholds.min_stress_pass` → `robustness.verdict=UNSTABLE`
- Else if `cycles_run < thresholds.min_cycles` → `robustness.verdict=INSUFFICIENT` and `gap_codes += e_sim_contract_incomplete`
- Else → `robustness.verdict=STABLE`

Note: `INSUFFICIENT` is fail-closed and MUST be treated as "do not use for verdict" by STEP 6.

---

## S6) High-Stakes Regime (policy-driven)

If `risk_class=HIGH`:
- thresholds MUST come from profile (no hardcoded “1000/0.999”),
- pass/fail is defined via U-OPMAP pass_criteria + stress suite,
- MUST include adversarial + drift scenarios,
- MUST log near-miss events as warnings tied to failure_modes.

---

## S7) Spillover & Borrowed Benefit Reporting

S MUST report:
- `spillover.cross_element_effects[]` when significant,
- `borrowed_benefit.is_borrowed=true` when:
  - persistent harm detected,
  - time-lag flip detected,
  - fragility debt detected under stress.

Debt channels MUST be recorded.

---

## S8) Emit Evidence

### S8.1 Required outputs
S MUST emit:
- `S-REPORT-*` as `E_sim`
- optional `S-CONSENSUS-*` as `E_cons`

### S8.2 E_sim minimal contract (MUST fields)
`E_sim` MUST include at minimum:

**Identification**
- `evidence_id`
- `class=E_sim`
- `sim_version`
- `profile_id/profile_version`
- `graph_version` (+ recommended `graph_hash`)
- `scenario_id`, `co_id`
- `scope_fit`, `time_window`

**Config binding**
- `alpha, tau, K_scale, E_scale`
- `thresholds: {min_cycles, min_stress_pass}`
- `eps_ab_delta`
- `critical_edges_used`

**Stress**
- `stress_suite: {cases[], pass_rate, pass_criteria_ref}`
- `stress_pass_rate` (MUST equal `stress_suite.pass_rate`)

**Signals/Findings**
- `element_signals` (raw + mapping params)
- `element_findings` (mark/magnitude/confidence)
- `spillover` and `borrowed_benefit` (if applicable)
- `borrowed_flag` (MUST equal `borrowed_benefit.is_borrowed`)

**Robustness**
- `builder_vs_critic_delta` (signed)
- `cycles_run` (int)
- `robustness: {verdict, notes}`

**Quality & Independence**
- `independence: {group_id, method_independent, data_independent, code_independent}`
- `quality: {credibility, reproducibility, limitations[], known_blindspots[]}`

**Audit hashes**
- `graph_hash`
- `config_hash`
- `code_hash`
- `intervention_hash`
- `stress_suite_hash`


### S8.3 STEP 6 compatibility (Required)
For downstream SVP-EVAL (STEP 6 v1.2), each emitted `E_sim` MUST include at top-level:
- `builder_vs_critic_delta`
- `stress_pass_rate`
- `cycles_run`
- `borrowed_flag` (optional but recommended)

If any of the first three are missing → treat as `e_sim_contract_incomplete`.

---

## S9) Misuse-safe constraints

- MUST NOT simulate outside scope: if `scope_fit.in_scope=false`, results MUST NOT be used downstream.
- MUST NOT emit “positive” evidence without Critic counterpart runs: missing B-side → `quality.credibility` downgrade + `robustness=INSUFFICIENT`.
- MUST record blindspots; empty blindspots MUST be explicitly declared as “blindspots_not_assessed”.

---


# S-REPORT-* → E_sim — template (minimal contract)
e_sim_template:
  evidence_id: "E_SIM_0001"
  class: "E_sim"
  protocol: "S"
  version: "1.0.2"
  sim_version: "SCS_SIM_v0.1"  # protocol-versioned simulation runner

  profile_id: "SCS_PROFILE_DEFAULT_v0.1"
  profile_version: "0.1"
  config_overrides: {}
  override_justifications: []
  applied_to_builder_and_critic: true

  graph_version: "CG_v0.1"
  graph_hash: "sha256:..."
  scenario_id: "SCN_..."
  co_id: "CO_..."

  scope_fit:
    in_scope: true
    notes: ""

  time_window:
    observed_from_utc: "YYYY-MM-DDThh:mm:ssZ"
    observed_to_utc: "YYYY-MM-DDThh:mm:ssZ"

  horizon_class: "medium"
  risk_class: "MED"
  flow_affecting: true

  thresholds:
    min_cycles: 500
    min_stress_pass: 0.90
  eps_ab_delta: 0.001
  cycles_run: 600

  normalization_params:
    alpha: 0.50
    tau: 0.05
    K_scale: 0.10
    E_scale: 0.10

  critical_edges_used: []

  stress_suite:
    suite_id: "STRESS_SUITE_DEFAULT_v0.1"
    cases: []
    pass_criteria_ref: "U-OPMAP-0001"
    pass_rate: 0.0

  stress_pass_rate: 0.0  # MUST equal stress_suite.pass_rate
    case_results:
      - case_id: "SE_NODE_AI"
        pass: true
        notes: ""
        post_stress_marks: { HUM: "0", AI: "+", BIO: "0", TEC: "0", SOC: "0" }

  element_signals:
    HUM: { S_e: 0.0, deltaK_norm: 0.0, deltaE_norm: 0.0, alpha: 0.5, tau: 0.05 }
    AI:  { S_e: 0.0, deltaK_norm: 0.0, deltaE_norm: 0.0, alpha: 0.5, tau: 0.05 }
    BIO: { S_e: 0.0, deltaK_norm: 0.0, deltaE_norm: 0.0, alpha: 0.5, tau: 0.05 }
    TEC: { S_e: 0.0, deltaK_norm: 0.0, deltaE_norm: 0.0, alpha: 0.5, tau: 0.05 }
    SOC: { S_e: 0.0, deltaK_norm: 0.0, deltaE_norm: 0.0, alpha: 0.5, tau: 0.05 }

  element_findings:
    HUM: { mark: "0", magnitude: 0.0, confidence: "weak", notes: "" }
    AI:  { mark: "0", magnitude: 0.0, confidence: "weak", notes: "" }
    BIO: { mark: "0", magnitude: 0.0, confidence: "weak", notes: "" }
    TEC: { mark: "0", magnitude: 0.0, confidence: "weak", notes: "" }
    SOC: { mark: "0", magnitude: 0.0, confidence: "weak", notes: "" }

  spillover:
    cross_element_effects: []

  borrowed_benefit:
    is_borrowed: false
    debt_channels: []

  borrowed_flag: false  # MUST equal borrowed_benefit.is_borrowed

  builder_vs_critic_delta: 0.0

  robustness:
    verdict: "INSUFFICIENT"   # STABLE|UNSTABLE|INSUFFICIENT
    notes: []

  independence:
    group_id: "SIMGROUP_01"
    method_independent: 0
    data_independent: 0
    code_independent: 0

  quality:
    credibility: 0.0
    reproducibility: 0.0
    limitations: []
    known_blindspots: ["blindspots_not_assessed"]

  audit:
    intervention_hash: "sha256:..."
    baseline_hash: "sha256:..."
    stress_suite_hash: "sha256:..."
    config_hash: "sha256:..."
    code_hash: "sha256:..."
    builder_seed: 0
    critic_seed: 0
    intervention_diff_summary: ""
    coverage_report:
      nodes_covered: []
      critical_edges_covered: []
      cases_total: 0
      cases_failed: 0
      cases_inconclusive: 0

  gap_codes: []  # optional; Appendix B codes

  compliance_summary:
    profile_bound: false
    coverage_ok: false
    overrides_ok: true
    reproducible_ok: false
    strict_mode_applied: false
