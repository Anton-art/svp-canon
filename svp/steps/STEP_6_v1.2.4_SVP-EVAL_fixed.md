# STEP 6 v1.2.4 — SVP-EVAL
**Status:** Normative  
**Role:** Aggregate evidence into `SystemDelta` and assign verdict  
`{TRUE_SYN | FALSE_SYN | HYPOTHESIS_SYN | UNKNOWN_SYN}` under **No-Guessing**.
**Revision:** v1.2.4 (2026-02-14) — adds Appendix D side outputs mapping (SideRecord) without changing Step 6 structure.

This step is the **only** step that assigns SVP verdict zones.  
Protocol U and Protocol S are upstream and MUST NOT assign truth/zone.

---

## 6.1 Inputs

### 6.1.1 Claim inputs
- `CO_candidate` (Claim Object)
- `claim_flow_affecting: bool` (from Step 4/5 routing or from Coupling Graph evaluation)
- `scope_ref` (U-SCOPE-*)

### 6.1.2 Evidence inputs
SVP-EVAL consumes the STEP 5 outputs (post-normalization):

- `ProofPlan` (policy_binding, horizon/risk, flow-affecting, coupling binding)
- `EvidenceSet` (records in ElementEvidenceRecord schema; **post-consolidation** provenance fields recommended)
- optional `ProvenanceClusterSummary` (clusters/sizes; if missing, Step 6 MAY recompute cluster_size from EvidenceSet)
- optional `EvidenceGapPlan` (Appendix B codes; may be empty)
- `ArtifactRefsBundle` (traceability pointers)

EvidenceSet records MAY include evidence classes:
`E_log`, `E_sim`, `E_ops`, `E_case`, `E_rep`, `E_fail`.

**Compatibility:** EvidenceSet SHOULD be `STEP_5 v1.2.2` or later.

### 6.1.3 Configuration inputs
- `horizon_class ∈ {short, medium, long}`
- `risk_class ∈ {LOW, MED, HIGH}`
- `stakes_profile_ref` (optional; if provided, MUST be policy-defined)

**Policy binding (Required):**
- `policy_config_ref` (e.g., `SVP-EVAL_PolicyPack_v0.1`)
- `policy_config_hash` (recommended)

**Module bindings (Required):**
- `ESS = Module A v0.1.2_fixed`
- `RTT = Module B v0.1.2_fixed`

Coupling resources:
- `coupling_graph` (Element-5 / edge definitions)
- optional `hard_constraints[]`, `tradeoff_allowed[]` (policy-defined)

---

## 6.2 Computability Gate (No-Guessing)

SVP-EVAL MUST fail closed when prerequisites are missing.

### 6.2.1 Simulation-required rule (flow-affecting)
SVP-EVAL MUST determine `simulation_required` deterministically via RTT triggers (Module B, B.8.1) using:
- `claim_flow_affecting`, `horizon_class`, `risk_class`, and optional `stakes_profile_ref`.

If `simulation_required=true` and **no** `E_sim` is present in EvidenceSet:
→ **verdict_core = UNKNOWN_SYN** and EvidenceGap `"simulation_required_missing"`.

### 6.2.2 Policy binding required
If `policy_config_ref` is missing, or required policy tables for ESS/RTT are missing:
→ **verdict_core = UNKNOWN_SYN** with EvidenceGap `"profile_binding_missing"`.

### 6.2.3 Strength scoring source-of-truth
Strength scoring MUST be delegated to **ESS (Module A)**; embedded default weights in Step 6 are not permitted.

---

## 6.3 Evidence Normalization (Required)

All evidence MUST be normalized into **ElementEvidenceRecord** objects before aggregation.

### 6.3.1 ElementEvidenceRecord schema
Each record `r` MUST include:

- `record_id: string`
- `evidence_class ∈ {E_log,E_sim,E_ops,E_case,E_rep,E_fail}`
- `element ∈ {HUM,AI,BIO,TEC,SOC}`
- `sign ∈ {+,0,-}` (effect direction on the element)
- `magnitude ∈ [0..1]` (normalized absolute effect size)
- `credibility ∈ [0..1]`
- `reproducibility ∈ [0..1]`
- `independence: {method_independent:int, data_independent:int, code_independent:int}`

**Provenance (Required identity):**
- `provenance.source_id: string | null` (REQUIRED; null only if EvidenceGap is emitted)
- `provenance.cluster_id: string | null` (REQUIRED; null only if EvidenceGap is emitted)
- `provenance.cluster_size: int | null` (POST-CONSOLIDATION; may be null pre-consolidation)
- `provenance.w_indep: float | null` (POST-CONSOLIDATION; may be null pre-consolidation)

**Simulation fields (REQUIRED iff evidence_class=E_sim):**
- `builder_vs_critic_delta: float` (signed)
- `stress_pass_rate: float ∈ [0..1]`
- `cycles_run: int ≥ 0`
- `borrowed_benefit.debt_channels[]` (may be empty list)

Optional:
- `robustness: {verdict: STABLE|UNSTABLE|INSUFFICIENT, reasons[]}?`
- `borrowed_flag: bool | null` (if provided by evidence, typically E_sim)
- `refs[]` (artifact ids / pointers)

### 6.3.2 No raw aggregation rule
No evidence MAY influence the verdict unless it is represented as a valid ElementEvidenceRecord.

If normalization fails (missing required fields):
→ **verdict_core = UNKNOWN_SYN** with EvidenceGap `"evidence_normalization_incomplete"` (or `"normalization_missing_fields"`).

If provenance identity cannot be derived deterministically (`source_id` or `cluster_id` null):
→ EvidenceGap `"provenance_missing_fields"` (Appendix B) and SVP-EVAL MUST fail closed by default:
- default: terminate `UNKNOWN_SYN`
- exception: only if `override_mechanism.allow_overrides=true` AND an explicit override object permits exclusion of the affected records.

---

## 6.4 Per-element Aggregation (Executable)

### 6.4.1 Strength scoring (ESS-bound, Required)
SVP-EVAL MUST compute record strength via **ESS (Module A v0.1.2_fixed)**.

For each record `r` in EvidenceSet:
1) Build `ESS_Input` with:
- `policy_config_ref`, `policy_config_hash`
- `horizon_class`, `risk_class`
- record fields: `evidence_class`, `credibility`, `reproducibility`, `magnitude`, `independence.*`
- provenance: `cluster_size` or `w_indep` (post-consolidation); if missing, follow Appendix B gap handling.
- for `E_sim`: `builder_vs_critic_delta`, `stress_pass_rate`, `cycles_run`
2) Call ESS → `strength ∈ [0..100]` plus breakdown (`Q`, `I_triplet`, `I_cluster`, `T`, and for `E_sim`: `S_stab`, `S_stress`, `S_cycles`).

Define normalized strength for aggregation:
- `s_norm(r) = strength(r) / 100.0`  (range [0..1])

**No-Guessing:** if ESS returns `BLOCKED_POLICY_MISSING` or required record fields are missing → Step 6 MUST terminate with `UNKNOWN_SYN` and appropriate EvidenceGap.

### 6.4.2 Aggregation by element
For each element `e`:

- `S_plus(e)  = Σ s_norm(r)` for records where `element=e and sign=+`
- `S_minus(e) = Σ s_norm(r)` for records where `element=e and sign=-`
- `S_zero(e)  = Σ s_norm(r)` for records where `element=e and sign=0`

Define:
- `ΔS(e) = S_plus(e) - S_minus(e)`
- `S_tot(e) = S_plus(e) + S_minus(e) + S_zero(e)`

### 6.4.3 Sign decision threshold (policy-bound)
Policy MUST define element direction thresholds:
- `policy.step6.theta_sign: {LOW: float, MED: float, HIGH: float}`

If `theta_sign` is missing → EvidenceGap `"profile_binding_missing"` and terminate `UNKNOWN_SYN`.

Decision per element `e`:
- If `ΔS(e) ≥ θ_sign[risk_class]` → `sign_e = '+'`
- Else if `-ΔS(e) ≥ θ_sign[risk_class]` → `sign_e = '-'`
- Else → `sign_e = '0'`

### 6.4.4 Confidence mapping (policy-bound)
Policy MUST define confidence mapping thresholds over `S_tot(e)` (normalized):
- `policy.step6.confidence_thresholds: {weak_lt: float, med_lt: float}`

Mapping:
- If `S_tot(e) < weak_lt` → `confidence_e = weak`
- Else if `S_tot(e) < med_lt` → `confidence_e = med`
- Else → `confidence_e = high`

**No-Guessing cap:** If `sign_e ∈ {+,-}` but `confidence_e == weak`, downgrade `sign_e → '0'` unless an `E_rep` or `E_fail` record supports that sign directly.

---

## 6.5 Robustness & Secondary-line Requirements

### 6.5.1 RTT gate for simulation-required cases (Required)
When `simulation_required=true`, SVP-EVAL MUST call **RTT (Module B v0.1.2_fixed)** once per claim evaluation.

RTT input is constructed from policy binding + claim context + an `E_sim` summary:
- `ab_delta = abs(builder_vs_critic_delta)`
- `stress_pass_rate` (from record)
- `cycles_run` (from record)
- `borrowed_benefit.debt_channels[]` (may be empty)
- optional `time_lag_ok` (only if simulation report provides it)

**Resampling (policy-enabled):**
If `policy.rtt.resampling.enabled=true`, Step 6 MUST compute and pass:
- `Ve_samples[]`: bootstrap samples of the aggregated simulation-support score `V_e_sim` under provenance-cluster resampling.
  - recommended definition: `V_e_sim = Σ s_norm(r)` over `E_sim` records after ESS scoring and cluster penalties.
- note: `V_e_sim` is the claim-level `V_e` expected by RTT (Module B, B.7.1).
- `influence[]`: jackknife influences computed by excluding one `cluster_id` at a time and measuring Δ(`V_e_sim`).

RTT returns `RTT_Output.verdict ∈ {STABLE, UNSTABLE, INSUFFICIENT}` plus deterministic `reasons[]`.

Verdict integration (fail-closed):
- If RTT verdict = `INSUFFICIENT` → `verdict_detail = UNKNOWN_SYN` and EvidenceGap `"robustness_inconclusive_on_key_elements"`.
- If RTT verdict = `UNSTABLE`      → `verdict_detail` cannot be `TRUE_SYN`; downstream MUST treat the result as `HYPOTHESIS_SYN` unless another blocker forces `UNKNOWN_SYN`.

---

## 6.6 Borrowed Benefit & Spillover Routing (Executable)

### 6.6.1 Borrowed benefit detection
Set `borrowed_benefit=true` if **any** holds:

(A) Some `E_sim` provides `borrowed_flag=true`, OR

(B) There exists a spillover path from a `'+'` element to a `'-'` element where:
- `confidence(to_element) ∈ {med,high}`, and
- at least one supporting record has `robustness.verdict==STABLE`, OR

(C) `E_sim` reports time-lag debt (negative flip at lag offsets) with `robustness=STABLE`.

### 6.6.2 Borrowed benefit policy
If `borrowed_benefit=true` and trade-offs are **not explicitly allowed** for this scenario:

- verdict MUST NOT be `TRUE_SYN`.
- SVP-EVAL MUST set:
  - `verdict_detail = UNKNOWN_SYN`
  - `verdict_core = UNKNOWN_SYN`
- SVP-EVAL MUST emit a `ScopeNarrowingProposal` stub and include debt channels (which elements carry the cost).

---

## 6.7 SystemDelta Assembly (Required)

SVP-EVAL MUST build a `SystemDelta` object that includes:

### 6.7.1 Per-element fields
For each element `e`:
- `sign_e`
- `confidence_e`
- `ΔS(e), S_plus(e), S_minus(e), S_tot(e)` (all based on normalized ESS strengths)
- `top_supporting_records[]` (record_id list; deterministic sort by contribution)
- `secondary_line_present: bool`
- `borrowed_channel_present: bool`
- `rtt_verdict: STABLE|UNSTABLE|INSUFFICIENT|none` (none if RTT not invoked)
- `rtt_reasons[]` (if invoked)

### 6.7.2 Global fields
- `policy_config_ref`, `policy_config_hash?`
- `modules: {ess_version, rtt_version}`
- `borrowed_benefit: {is_borrowed: bool, channels[]}`
- `tradeoffs_policy_ref` (or `tradeoffs_not_allowed_default`)
- `computability_gate_result`
- `robustness_summary` over invoked RTT outputs (min verdict under order STABLE>UNSTABLE>INSUFFICIENT, or `none`)
- `provenance_cluster_summary_ref?`
- `notes[]`

---

## 6.8 Axis-5 Downgrade-only Check

Axis-5 MAY only:
- downgrade signs (`'+'→'0'`, `'-'→'0'`)
- downgrade confidence levels

Axis-5 MUST NOT upgrade any sign.

Axis-5 MUST cite downgrade trigger(s) as EvidenceGap codes (Appendix B), e.g.:
- `borrowed_benefit_blocking_true`
- `secondary_line_missing_evidence` / `secondary_line_insufficient_independence`
- `robustness_inconclusive_on_key_elements`
- `scope_mismatch`
- `witness_attestation_missing`

---

## 6.9 Final Verdict Rules (Executable)

SVP-EVAL exposes two verdict layers:
- `verdict_detail ∈ {TRUE_SYN, FALSE_SYN, HYPOTHESIS_SYN, UNKNOWN_SYN}`
- `verdict_core ∈ {TRUE_SYN, FALSE_SYN, UNKNOWN_SYN}`

Mapping:
- `verdict_core = UNKNOWN_SYN` whenever `verdict_detail ∈ {HYPOTHESIS_SYN, UNKNOWN_SYN}`.

### 6.9.1 TRUE_SYN
Verdict MAY be `TRUE_SYN` only if **all** hold:

1) Computability gate passed  
2) If `simulation_required=true` → at least one `E_sim` present AND RTT verdict for required checks is not `INSUFFICIENT`  
3) `borrowed_benefit == false`, OR (borrowed true AND trade-offs explicitly allowed AND debt is outside scope)  
4) For all elements with `sign='-'`, either:
   - explicitly outside scope, OR
   - explicitly allowed trade-off with constraints satisfied  
5) At least one element has `sign='+'`  
6) No element in `hard_constraints[]` has `sign='-'`  
7) For medium/long horizons: SOC/BIO secondary-line rule satisfied if they are non-zero signs

### 6.9.2 FALSE_SYN
Verdict is `FALSE_SYN` if **any** holds:

- `hard_constraints[]` violated (any hard-constraint element has `sign='-'`), OR
- `harm_robust=true`, where:

`harm_robust` means:
- there exists element `e` with `sign_e='-'` and `confidence_e ∈ {med,high}`, AND
- either:
  - there exists `E_fail` or `E_rep` supporting the negative, OR
  - `simulation_required=true` and RTT verdict is `STABLE` while `'-'` persists under stress/cycles, AND
- harm is within scope boundaries (`scope_fit.in_scope=true` for that regime)

### 6.9.3 HYPOTHESIS_SYN
`verdict_detail = HYPOTHESIS_SYN` if:

- computability gate passed, AND
- at least one `'+'` exists, AND
- no robust harm triggers `FALSE_SYN`, BUT one of:
  - key confidence is `weak`
  - secondary line missing (required)
  - RTT verdict is `UNSTABLE` for required checks
  - borrowed benefit uncertain due to insufficient evidence

### 6.9.4 UNKNOWN_SYN
`verdict_detail = UNKNOWN_SYN` if any holds:

- computability gate fails, OR
- evidence normalization incomplete, OR
- sign thresholds not met (all elements are `'0'`), OR
- `borrowed_benefit=true` and trade-offs not explicitly allowed, OR
- policy binding missing for required tables (ESS/RTT)

### 6.9.5 Required outputs
SVP-EVAL MUST emit:
- `verdict_detail` and `verdict_core`
- `SystemDelta`
- `EvidenceGapPlan` (if `verdict_core=UNKNOWN_SYN` or any blocking gaps exist)
- `ScopeNarrowingProposal stub` if borrowed benefit blocks `TRUE_SYN`

---

## 6.10 Scope Narrowing Procedure (Required when proposed)

If SVP-EVAL proposes scope narrowing, it MUST output `ScopeNarrowingProposal` with:

- `proposal_id`
- `original_scope_ref`
- `excluded_regimes[]`
- `justification`
- `evidence_refs[]`
- `residual_risks[]`
- `validity_checks[]`:
  - exclusions do not contradict the claim statement
  - excluded regimes are supported by evidence as debt regions
  - new scope remains meaningful (non-trivial)

**Decision authority note:**  
Human role remains **WITNESS** (attest artifacts/constraints), not a truth judge. Acceptance of narrowing is governance outside U/S/SVP-EVAL.

---

## 6.11 Default Trade-offs Policy Binding

SVP-EVAL MUST read:
- `hard_constraints[]`
- `tradeoff_allowed[]`

from Coupling Graph / policy profile.

If absent:
- treat as `"tradeoffs_not_allowed_default"` (conservative).

---

---

## 6.12 Side outputs (Appendix D) (Required)

Step 6 MUST emit side outputs per **Appendix D — Side Outputs & Banks Contract v0.1**:
- `Appendix_D_SideOutputs_Banks_Contract_v0.1.md`
- SideRecord template example: `Appendix_D_SideRecord_Template_v0.1.yaml`

**Constraint:** this section adds an additional output channel (`side_records[]`) and mapping. It does **not** change Step 6 verdict logic.

### 6.12.1 When to emit SideRecord
For each evaluated claim/hypothesis, Step 6 MUST emit a SideRecord if ANY is true:
- final verdict is `UNKNOWN_SYN` (not enough support to assert TRUE/FALSE), OR
- verdict is `INSUFFICIENT` due to gaps (provenance/coverage/simulation-required), OR
- verdict is `REJECTED_*` due to policy constraints at evaluation time.

If verdict is final TRUE (forward terminal output), emitting a SideRecord is OPTIONAL (traceability).

### 6.12.2 SideRecord mapping (normative)

- `step_id = STEP6`
- `input_ref` MUST link to:
  - the Step 5 EvidenceSet/ProofPlan record id, and
  - claim_id (or claim pointer).
- `normalized_claim` MUST match the claim evaluated in Step 6.

**Status mapping**
- If Step 6 verdict is `UNKNOWN_SYN` → `status = UNKNOWN_SYN`
- If Step 6 verdict is `INSUFFICIENT` (any gap-based block) → `status = INSUFFICIENT` and `reason_codes` MUST include Appendix B codes.
- If Step 6 explicitly rejects due to policy hard blocks (rare) → map to:
  - `REJECTED_CHAOS` (if chaos re-check is the reason), else `INSUFFICIENT`.

Legacy status `inconclusive` MUST NOT be used.

**scores{} mapping (minimum)**
`scores` MUST include:
- `truth_score` (or equivalent final scoring field, if present)
- `evidence_strength` (from Module A / Step 5 artifacts)
- `robustness_score` (from Module B / RTT mapping)
- `w_indep` (if available)
- `cluster_size` (if available)
- `gap_penalty` (0 if no gaps)
- `confidence_band` (if present; else omit)

**reason_codes[] mapping**
- If `status=INSUFFICIENT`, `reason_codes` MUST include the exact Appendix B codes from the EvidenceGapPlan and any adapter provenance gaps (Appendix C).
- If `status=UNKNOWN_SYN`, `reason_codes` SHOULD include:
  - `insufficient_support` (or closest existing code in PolicyPack), AND
  - any dominant conflict reason labels used in Step 6.

**distance_to_pass mapping**
- If `UNKNOWN_SYN`:
  - `distance_to_pass` MUST represent the nearest boundary to a decisive zone (TRUE or FALSE) using the primary `truth_score`:
    - `{kind:numeric, metric:truth_score, threshold:<nearest_decisive_threshold>, value:truth_score, margin:(value - threshold)}`
- If `INSUFFICIENT` (gap-based):
  - `distance_to_pass = {kind: numeric, metric: gap_count_total, threshold: 0, value: gap_count_total, margin: -gap_count_total}`
  - If gap count is not available, use categorical miss: `{kind: categorical, miss: INSUFFICIENT_GAPS}`.

**recommended_protocol mapping (default)**
- For `INSUFFICIENT` with simulation-required gaps → `recommended_protocol = SIMULATION` (Protocol S).
- For `INSUFFICIENT` with provenance gaps → `recommended_protocol = EVIDENCE` (or `PARTNER_DEV` if adapter repair is needed).
- For `UNKNOWN_SYN` with high novelty but low evidence → `recommended_protocol = EVIDENCE` (collect support).
- For `UNKNOWN_SYN` with conceptual ambiguity/logic conflicts → `recommended_protocol = FORMALIZE` (Protocol U) or `PARTNER_DEV`.

### 6.12.3 Reopen triggers (minimum)
Step 6 SideRecords SHOULD include at least:
- `HUMAN_CHALLENGE`
- `NEW_SOURCE_CLUSTER`
- `POLICY_CHANGED`
- `COUPLING_GRAPH_CHANGED`

---

