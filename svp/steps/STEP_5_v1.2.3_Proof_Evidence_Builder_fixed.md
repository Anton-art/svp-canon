# STEP 5 v1.2.3 — Proof / Evidence Builder
**Status:** Normative  
**Revision:** v1.2.3 (2026-02-14) — adds Appendix D side outputs mapping (SideRecord) without changing Step 5 structure.
**Purpose:** Build **evidence artifacts** (`E_*`) for downstream **STEP 6 (SVP-EVAL)**.  
**Core constraint:** STEP 5 is **evidence-only** and MUST NOT assign truth/zones.

---

## 5.1 Core Rules (Misuse-safe)

### 5.1.1 Evidence-only
STEP 5 MUST NOT output:
- `TRUE_SYN / FALSE_SYN / HYPOTHESIS_SYN / UNKNOWN_SYN`
- final truth/zones for elements or for the system

STEP 5 MUST output:
- evidence artifacts (`E_log, E_sim, E_ops, E_case, E_rep, E_fail`)
- a computable plan and/or gap plan if evidence cannot be produced

### 5.1.2 No-Guessing
If required evidence cannot be built with declared scope/definitions/coverage, STEP 5 MUST:
- stop as `BLOCKED_EVIDENCE_PLAN_INCOMPLETE`
- emit an `EvidenceGapPlan` describing what is missing and why it matters.

### 5.1.3 Human role
Human role in STEP 5 is **WITNESS**:
- attests artifact integrity, scope bounds, hashes, ethical admissibility constraints
- does **not** decide truth/verdict.

---

## 5.2 Inputs

### 5.2.1 Claim inputs
- `CO_candidate`
- `U-artifacts` (if available / required):
  - `U-SCOPE-*`
  - `U-TERMS-*`
  - `U-ASSUMPTIONS-*`
  - `U-FALSIFIERS-*`
  - `U-OPMAP-*` (required if simulation is planned)

### 5.2.2 Evidence inputs (optional pre-existing)
- existing `E_*` artifacts from prior runs, field data, ops logs, replications.

### 5.2.3 Policy inputs
- `horizon_class ∈ {short, medium, long}`
- `risk_class ∈ {LOW, MED, HIGH}`
- active simulation profile/policy (if S is planned): thresholds, eps, stress rules

### 5.2.4 Coupling graph binding (required)
STEP 5 MUST bind to a Coupling Graph version:
- `coupling_graph_version`
- `coupling_graph_hash` (if available)

STEP 5 MUST compute `claim_flow_affecting` via Coupling Graph FA rules (not heuristics).

---

## 5.3 Outputs (Artifacts)

STEP 5 MUST output:

1) **ProofPlan**  
Declares which evidence to produce, how, with what coverage and stop rules.

2) **EvidenceSet**  
A consolidated set of evidence lines, each **ElementEvidenceRecord-compatible** (see 5.6.3).

3) **EvidenceGapPlan** (if needed)  
Machine-readable list of missing fields / missing lines / missing coverage.

4) Evidence artifacts themselves:
- `E_log` (logic/derivation evidence)
- `E_sim` (simulation evidence, evidence-only)
- `E_ops` (telemetry/ops evidence)
- `E_case` (case evidence)
- `E_rep` (replication evidence)
- `E_fail` (strong negative / falsification evidence)

Every artifact MUST declare:
- `evidence_class`
- element coverage (`HUM, AI, BIO, TEC, SOC`) where applicable
- references to upstream scope/terms and to downstream adapters if any.

---

## 5.4 Routing (U-only vs U→S)

### 5.4.1 Flow-affecting routing is authoritative
If `claim_flow_affecting=true`:
- STEP 5 MUST include `S_SIMULATION` in ProofPlan,
- and MUST plan stress coverage for critical edges (5.5.3),
- and MUST ensure `U-OPMAP-*` exists (or emit gap: `missing_opmap`).

If `claim_flow_affecting=false`:
- STEP 5 MAY proceed with U-only evidence (E_log/E_case/E_ops/E_rep/E_fail),
- provided evidence is normalizable (5.6.3).

---

## 5.5 ProofPlan (Required)

### 5.5.1 Required fields
ProofPlan MUST include:
- `co_id`
- `coupling_graph_version`, `coupling_graph_hash?`
- `horizon_class`, `risk_class`
- `claim_flow_affecting`
- selected protocols: `{U, S}` (S required if flow-affecting)
- expected evidence classes and target elements
- stop conditions
- validation checks required before emission to STEP 6

### 5.5.2 No hardcoded high-stakes constants
STEP 5 MUST NOT hardcode constants like “1000 cycles” or “99.9% success”.
Instead, ProofPlan MUST reference **policy/profile** parameters:
- `S_config.thresholds.min_cycles`
- `S_config.thresholds.min_stress_pass`
- `S_config.eps_ab_delta` (or profile stability eps)
- any strict-mode requirements

### 5.5.3 Critical edges + stress suite planning (flow-affecting)
If `claim_flow_affecting=true`, ProofPlan MUST include:
- `critical_edges_used` derived by Coupling Graph default rule + intervention closure
- `stress_suite_plan`:
  - node coverage: all 5 elements at least once
  - edge coverage: each critical edge at least once
  - mixed shocks: required for flow-affecting

Failure to plan this coverage MUST be recorded as EvidenceGap.

### 5.5.4 Secondary-line planning for SOC/BIO (medium/long)
If `horizon_class ∈ {medium,long}` AND ProofPlan expects any non-zero effect for `SOC` or `BIO`:
- ProofPlan MUST require a **secondary independent line**:
  - evidence_class ∈ `{E_ops, E_case, E_rep, E_fail}`
  - independence constraint: `data_independent ≥ 1` AND (`method_independent ≥ 1` OR `code_independent ≥ 1`)

If not planned → EvidenceGap `"secondary_line_missing_plan"`.

---

## 5.6 Evidence Quality & Normalization

### 5.6.1 EvidenceHeader requirements
For each evidence artifact, STEP 5 MUST provide:
- `credibility ∈ [0..1]`
- `reproducibility ∈ [0..1]`
- `independence: {method_independent:int, data_independent:int, code_independent:int}`
  - booleans are allowed only if an adapter deterministically maps them to ints
- `scope_fit: {in_scope:bool, notes}`

### 5.6.2 Magnitude requirement
Wherever STEP 5 emits a sign/mark for an element, it MUST also emit:
- `magnitude ∈ [0..1]` (normalized absolute effect size)

Marks alone are insufficient for STEP 6 aggregation.

### 5.6.3 ElementEvidenceRecord compatibility (Normalization contract)
STEP 5 MUST ensure each evidence “line” can be expressed as:

- `record_id`
- `evidence_class`
- `element`
- `sign ∈ {+,0,-}`
- `magnitude ∈ [0..1]`
- `credibility ∈ [0..1]`
- `reproducibility ∈ [0..1]`
- `independence {method_independent:int, data_independent:int, code_independent:int}`
- `robustness.verdict` (if evidence_class=E_sim: REQUIRED; else optional)
- `borrowed_flag` (if provided by E_sim; else null)
- `provenance.source_id` (REQUIRED)
- `provenance.cluster_id` (REQUIRED)
- `provenance.cluster_size` (POST-CONSOLIDATION; may be null pre-consolidation)
- `provenance.w_indep` (POST-CONSOLIDATION; may be null pre-consolidation)
- `refs[]`

**No-Guessing:** If any REQUIRED field cannot be provided, STEP 5 MUST emit EvidenceGapPlan item:
- `"normalization_missing_fields"` (with exact field list), OR
- `"provenance_missing_fields"` (with exact field list) when provenance fields cannot be derived deterministically.

---

### 5.6.4 Provenance & Source Independence (Normative)
STEP 5 MUST set provenance fields deterministically via adapters (see **Appendix C — Adapter Rules v0.1.2_fixed**):

- `provenance.source_id` MUST be a stable identifier derived from the source (e.g., `doi:...`, `url:...`, `commit:...`, `hash:sha256:...`).
- `provenance.cluster_id` MUST be derived from policy-defined cluster key fields (Appendix C).
- `provenance.cluster_size` and `provenance.w_indep` MUST be computed during consolidation (5.8.3).

If adapter cannot derive `source_id` or `cluster_id` without guessing, it MUST:
- set them to null, and
- emit an EvidenceGapPlan item `provenance_missing_fields` listing missing fields.

## 5.7 Simulation Evidence Builder (Protocol S integration)

### 5.7.1 Mirror protocol A/B
If simulation is run, STEP 5 MUST require:
- Builder run (A)
- Critic run (B)
under the same declared profile bindings and normalization params.

### 5.7.2 Replace fixed stability with profile parameters
Stability/robustness MUST be computed via:
- `builder_vs_critic_delta (ab_delta)` compared to `eps_ab_delta` (profile)
- `stress_suite.pass_rate` compared to `min_stress_pass` (profile)
- cycles compared to `min_cycles` (profile)

### 5.7.3 E_sim minimal contract (MUST fields)
If STEP 5 emits `E_sim`, it MUST include at minimum (compatible with **RTT v0.1.2**):

**Required (RTT-required fields):**
- `robustness.verdict ∈ {STABLE, UNSTABLE, INSUFFICIENT}`
- `ab_delta` (MUST; absolute builder-vs-critic disagreement)
  - `ab_delta = abs(builder_vs_critic_delta)` if raw delta is present
- `stress_pass_rate ∈ [0..1]` (MUST)
- `cycles_run` (MUST; integer ≥ 0)
- `borrowed_benefit.debt_channels[]` (MUST; may be empty)

**Required for flow-affecting claims:**
- `critical_edges_used[]` (MUST if `claim_flow_affecting=true`)

**Traceability (recommended; may be REQUIRED by profile/policy):**
- `builder_vs_critic_delta` (raw signed delta)
- `stress_suite` (recommended; use when per-case traceability is required):
  - `cases[]`
  - `case_results[]` with `{case_id, passed:bool}` (notes optional)
- audit bindings/hashes if available (recommended):
  - `coupling_graph_hash`, `sim_config_hash`, `code_hash`, `intervention_hash`, `stress_suite_hash`

**Policy binding:** `eps`, `min_stress_pass`, `min_cycles`, and any strict-mode requirements MUST come from policy/profile, not hardcoded in STEP 5.

If any REQUIRED field is missing → EvidenceGap `"e_sim_contract_incomplete"` (with exact missing fields).

---

## 5.8 Evidence Consolidation (EvidenceSet)

### 5.8.1 Consolidation rules
STEP 5 MUST consolidate evidence into an EvidenceSet without losing normalization fields.

Lossy consolidation is forbidden:
- do not drop magnitude,
- do not drop independence,
- do not drop robustness fields,
- do not drop refs.

### 5.8.2 EvidenceSet must preserve element coverage
EvidenceSet MUST explicitly list which elements have supporting evidence lines and which are empty.


### 5.8.3 Provenance consolidation (clusters) (Normative)
During consolidation, STEP 5 MUST:

1) Group all EvidenceSet records by `provenance.cluster_id` (null cluster_id records MUST be treated as gaps).
2) Compute `provenance.cluster_size = size(cluster)` and write it into every record in the cluster.
3) Compute independence penalty weight:
- `alpha = policy.provenance.cluster_alpha`
- `w_indep = 1 / (cluster_size ^ alpha)`
- rounding MUST be deterministic (round-half-up to 3 decimals) if the value is stored as decimal.
4) Emit `ProvenanceClusterSummary` artifact listing clusters and member record_ids.

If any record lacks `source_id` or `cluster_id`, STEP 5 MUST emit `EvidenceGapPlan.provenance_missing_fields` and MAY stop with `BLOCKED_EVIDENCE_PLAN_INCOMPLETE` depending on policy strictness.
---

## 5.9 Preconditions / Precheck (Executable)

Before emitting output to STEP 6, STEP 5 MUST validate:

(A) **Normalization completeness**  
All evidence lines satisfy 5.6.3.

(B) **Flow-affecting compliance**  
If `claim_flow_affecting=true`, `E_sim` is present or planned.

(C) **SOC/BIO robustness for medium/long**  
If using E_sim to justify non-zero SOC/BIO:
- at least one STABLE E_sim is present/planned.

(D) **Secondary line**  
If required (5.5.4), secondary independent line is present/planned.

(E) **Borrowed benefit readiness**  
If flow-affecting:
- borrowed_flag/spillover fields are present/planned (so Step 6 can route borrowed benefit).

If any check fails:
- STOP with `BLOCKED_EVIDENCE_PLAN_INCOMPLETE`
- emit `EvidenceGapPlan`.

---

## 5.10 Emission to STEP 6 (Required outputs)

STEP 5 MUST output (at minimum):

1) **ProofPlan** (with coupling binding, horizon/risk, flow-affecting, coverage plans, stop conditions)  
2) **EvidenceSet** (ElementEvidenceRecord-compatible records, post-consolidation provenance fields populated)  
3) **EvidenceGapPlan** (may be empty; if non-empty must include deterministic codes from Appendix B)  
4) **ArtifactRefsBundle** (upstream refs + produced evidence ids + downstream targets)  
5) **PrecheckReport** (PASS/FAIL + reasons; required when blocking)  

STEP 5 SHOULD output when applicable:
- **ProvenanceClusterSummary** (clusters, sizes, member ids)  

**No-Truth constraint:** none of these outputs may assert final truth/zones.

---

## 5.11 Exit statuses (Normative)
STEP 5 MUST terminate in one of:
- `OK_EVIDENCESET_READY`
- `OK_EVIDENCE_PLAN_EMITTED` (when gaps exist but plan is actionable)
- `BLOCKED_EVIDENCE_PLAN_INCOMPLETE`

All non-OK exits MUST include an EvidenceGapPlan.

---

## 5.12 Side outputs (Appendix D) (Required)

Step 5 MUST emit side outputs per **Appendix D — Side Outputs & Banks Contract v0.1**:
- `Appendix_D_SideOutputs_Banks_Contract_v0.1.md`
- SideRecord template example: `Appendix_D_SideRecord_Template_v0.1.yaml`

**Constraint:** this section adds an additional output channel (`side_records[]`) and mapping. It does **not** modify Step 5’s evidence-only rule set.

### 5.12.1 When to emit SideRecord
For each ProofPlan/EvidenceSet attempt, Step 5 MUST emit a SideRecord if ANY is true:
- `EvidenceGapPlan` is non-empty (any gaps), OR
- exit status is `BLOCKED_EVIDENCE_PLAN_INCOMPLETE`, OR
- Precheck fails (`PrecheckReport = FAIL`).

If exit status is `OK_EVIDENCESET_READY` with empty EvidenceGapPlan, emitting a SideRecord is OPTIONAL (traceability).

### 5.12.2 SideRecord mapping (normative)
For Step 5, SideRecord MUST be produced with:

- `step_id = STEP5`
- `input_ref` MUST link to:
  - `claim_id` (or claim pointer from Step 4), and
  - the `ProvenanceClusterSummary` / cluster bindings when available (Appendix C fields).
- `normalized_claim` MUST equal the normalized claim carried into Step 5.

**Status mapping**
- If `EvidenceGapPlan` non-empty OR `PrecheckReport=FAIL` OR exit=`BLOCKED_EVIDENCE_PLAN_INCOMPLETE`  
  → `status = INSUFFICIENT`
- Step 5 MUST NOT emit `UNKNOWN_SYN` (reserved for Step 6), and MUST NOT emit final truth statuses.

**scores{} mapping (minimum)**
`scores` MUST include:
- `precheck_pass` (bool as 0/1)
- `gap_count_total`
- `gap_count_provenance` (count of provenance-related gaps)
- `gap_count_coverage` (count of coverage-related gaps)
- `gap_count_simulation` (count of simulation-required / e_sim gaps)
- `evidence_lines_count` (EvidenceSet line count, if any)
- `cluster_count` and `mean_cluster_size` (if ProvenanceClusterSummary is present)

If Step 5 computed `w_indep` (5.8.3), it SHOULD add:
- `w_indep_min`, `w_indep_mean`

**reason_codes[] mapping**
- If `EvidenceGapPlan` is non-empty:
  - `reason_codes` MUST include the exact Appendix B codes present in `EvidenceGapPlan`.
- If `PrecheckReport=FAIL` but EvidenceGapPlan is empty (should not happen):
  - `reason_codes` MUST include `precheck_failed_without_gap_plan` (implementation bug signal).

**distance_to_pass mapping**
- If blocked or precheck fail:
  - `distance_to_pass = {kind: categorical, miss: <exit_status>}`
- If gaps exist but exit is OK (`OK_EVIDENCE_PLAN_EMITTED`):
  - `distance_to_pass = {kind: numeric, metric: gap_count_total, threshold: 0, value: gap_count_total, margin: -gap_count_total}`

**recommended_protocol mapping (default)**
- If gap set includes `simulation_required` or any `e_sim_*` contract gaps → `recommended_protocol = SIMULATION` (Protocol S)
- Else if gaps are mostly provenance (`provenance_missing_fields`, cluster binding gaps) → `recommended_protocol = EVIDENCE` (fix sources/provenance) and MAY be `PARTNER_DEV` for adapter repair.
- Else (coverage/secondary line/flow-affecting readiness) → `recommended_protocol = EVIDENCE`.

### 5.12.3 Reopen triggers (minimum)
Step 5 SideRecords SHOULD include at least:
- `HUMAN_CHALLENGE`
- `NEW_SOURCE_CLUSTER` (cluster_size or w_indep improves)
- `POLICY_CHANGED` (strictness/threshold changes)
- `COUPLING_GRAPH_CHANGED` (if ProofPlan is coupling-bound)

---

