# SVP v0.2.3 — Syntropy Verdict Protocol
**Status:** Normative (Engine Spec)  
**Compatibility:** v0.2.x
**Revision:** v0.2.3 (2026-02-13) — stykovka alignment to SVP-EVAL canon; legacy inconclusive status → INSUFFICIENT; E_sim field contract updated.
Core idea: “Syntropy-truth” определяется не по соответствию фактам, а по вкладу в синтропию системы из 5 элементов {HUM, AI, BIO, TEC, SOC} при заданном scope/домене и доказательной базе.
## 0. Design Invariants (MUST)

## 0.1 SVP-EVAL canon mapping (Normative)
SVP v0.2.3 is executed through the SVP-EVAL pipeline:

- STEP 1 v1.1.2 — intake + claim extraction + RunContext
- STEP 2 v1.1.2 — Stage A cheap chaos score
- STEP 3 v1.1.2 — Stage B main chaos index + step4_handoff
- STEP 4 v1.1.4 — novelty/connectedness + near-dup + orphan routing
- STEP 5 v1.2.2 — proof/evidence builder (ElementEvidenceRecord)
- STEP 6 v1.2.3 — SVP-EVAL verdict assembly

Modules / protocols:
- Module A (ESS) v0.1.2 — element strength scoring
- Module B (RTT) v0.1.2 — robustness & truth-triggering
- Protocol U v1.0.2 — orphan consensus (witness-only)
- Protocol S v1.0.2 — simulation evidence (E_sim)

Policy:
- SVP-EVAL Policy Pack v0.1 (updated_fixed_step6)

No TRUE/FALSE without Compute-Readiness. Если условия вычислимости не выполнены → только UNKNOWN_SYN.
No-Guessing. Нельзя ставить +/− по элементу без минимального профиля evidence для этого элемента → принудительно 0.
Simulation is evidence-only. E_sim никогда не является “каноном истины”; он лишь линия evidence.
Axis-5 downgrade-only. Axis-5 может только сузить scope или понизить TRUE→UNKNOWN.
Human/Agent = Witness of artifacts/scope. Люди/агенты не “судьи истины”, только подтверждают артефакты, границы, выбор.
## 1. Objects
### 1.1 Elements
ELEM = {HUM, AI, BIO, TEC, SOC}
### 1.2 Verdicts
syntropy_verdict ∈ { TRUE_SYN, FALSE_SYN, UNKNOWN_SYN }
### 1.3 Zones (“Crystal”)
syntropy_zone ∈ { TRUTH_ZONE, LIE_ZONE, HYPOTHESIS_ZONE }
Mapping (MUST):
TRUE_SYN → TRUTH_ZONE
FALSE_SYN → LIE_ZONE
UNKNOWN_SYN → HYPOTHESIS_ZONE
### 1.4 Domains and Horizons
domain ∈ {D0, D1, D2, D3, D4_*, D5, UNKNOWN}
horizon_class (MUST):
D0, D1 → short
D2, D3 → medium
D4_*, D5 → long
UNKNOWN → UNKNOWN_SYN (via gate)
### 1.5 CO — Claim Object (Input to SVP-EVAL)
CO MUST include:
co_id
claim_text_norm
domain
scope.boundaries (object, not empty)
quality.noise_flag
impact_evidence_refs[] (evidence IDs)
witness_refs[] (witness records)
limitations[] (optional)
risk_class ∈ {LOW, MED, HIGH}
flow_affecting ∈ {true,false}
graph_version (required if flow_affecting=true)
## 2. Evidence Canon
### 2.1 Allowed evidence classes
E_class ∈ { E_ops, E_case, E_rep, E_fail, E_sim, E_cons, E_bio }
Any other evidence class MUST be rejected as invalid for SVP.
### 2.2 Evidence strength ordering (for aggregation only)
E_fail > E_rep > E_ops > E_case > E_cons > E_sim
This ordering:
influences conflict resolution and confidence,
does not override No-Guessing.
### 2.3 Common evidence header (MUST fields)
Every evidence artifact MUST include:
evidence_id
class
evidence_version
created_utc
scope_fit: { in_scope: bool, notes? }
time_window: { observed_from_utc, observed_to_utc } (or explicit null + justification)
independence: { group_id?, method_independent, data_independent, code_independent }
quality: { credibility [0..1], reproducibility [0..1], limitations[] }
element_findings (when applicable): per element {mark, magnitude, confidence, note}
Out-of-scope evidence (scope_fit.in_scope=false) MUST NOT contribute to marks/robustness.
### 2.4 E_sim required fields (coupling-aware)
If class=E_sim, evidence MUST also include:
schema_version (string)
sim_run_id (string)
created_utc (datetime)
verdict ∈ {STABLE, UNSTABLE, INSUFFICIENT}
graph_version
builder_vs_critic_delta (numeric)
stress_suite.pass_rate (0..1)
stress_pass_rate (0..1)  # alias: equals stress_suite.pass_rate
cycles_run (int)
spillover summary:
cross_element_effects[] with path info
borrowed_benefit block:
borrowed_flag (bool)  # alias: equals borrowed_benefit.is_borrowed
is_borrowed: bool
debt_channels[] (at least empty list)
gap_codes[] (list of Appendix B codes)
hash_bindings block (policy/scenario hashes)
robustness: { verdict: STABLE|UNSTABLE|INSUFFICIENT, eps, thresholds, notes[] }
## 3. No-Guessing Profiles (Element Evidence Profiles)
### 3.1 Rule NG-1 (MUST)
For each element e, if the minimal evidence profile for e is not met, then ΔE(e) MUST = 0 even if some evidence suggests +/-.
### 3.2 Coupling limitation (MUST)
Even if E_sim exists:
SOC MUST NOT be set to +/- unless its profile is met by additional non-sim evidence (E_case/E_rep/E_cons etc.).
Under strict policy, BIO MUST NOT be set to +/- by E_sim alone.
(Profiles are configured per your SVP-02 table; v0.2.2 does not redefine them, it enforces them.)
## 4. Flow-Affecting Claims and Coupling Evidence
### 4.1 Definition
A claim is flow_affecting=true iff it:
modifies inter-element transfers/couplings, OR
asserts cross-element causal improvement “X ⇒ Y” where X≠Y, OR
is a systemic regulator claim (noise/coherence/efficiency) expected to propagate.
### 4.2 Coupling Evidence Gate (MUST)
If flow_affecting=true, then impact_evidence_refs[] MUST include at least one valid E_sim for the same scope and graph_version.
If not satisfied → UNKNOWN_SYN with reason MISSING_COUPLING_EVIDENCE.
## 5. SystemDelta (Internal Aggregation Object)
### 5.1 Purpose
SVP-EVAL MUST compute SystemDelta from evidence first, then derive ΔE(element) subject to No-Guessing.
### 5.2 SystemDelta schema (MUST fields)
delta_node[element] = { sign: +|0|-, magnitude: 0..1, confidence: weak|med|high }
delta_edge[src→dst] (optional but recommended for trace)
spillover_map[] (derived from E_sim and/or empirical)
borrowed_benefit: { is_borrowed: bool, debt_channels[] }
robustness_summary: { stable: bool, stress_pass_rate, cycles, ab_delta, time_lag_ok, verdict }
aggregation_notes[]
## 6. Evidence Strength Scoring and Robustness Thresholds
SVP v0.2.2 depends on two normative modules:
ESS v0.1 — Evidence Strength Scoring
RTT v0.1 — Robustness Threshold Table
Engine MUST implement ESS+RTT exactly as configured for:
converting evidence → strength and confidence,
vote aggregation thresholds Θ,
E_sim minima (cycles/stress/eps) per horizon/risk,
robust harm/beneﬁt requirements.
If ESS/RTT are not available → evaluator MUST return UNKNOWN_SYN with reason RESOURCE_BLOCKED.
## 7. SVP-EVAL(CO) — Evaluation Pipeline
Stage 1 — Horizon assignment (MUST)
Compute horizon_class from domain.
Stage 2 — Computability Gate (MUST)
If any condition fails → UNKNOWN_SYN and emit EvidenceGapPlan.
Gate conditions:
domain != UNKNOWN
scope.boundaries defined and non-empty
quality.noise_flag != NOISE
impact_evidence_refs[] not empty
If flow_affecting=true → CG-SIM satisfied (E_sim present, valid, in-scope)
Stage 3 — Evidence validation (MUST)
each evidence referenced exists and is in allowed class
scope_fit.in_scope=true required to contribute
must-fields per class present (including E_sim extra fields)
If validation fails: UNKNOWN_SYN, reason OUT_OF_SCOPE_EVIDENCE_ONLY or RESOURCE_BLOCKED, and emit EvidenceGapPlan.
Stage 4 — Build SystemDelta (MUST)
Using ESS+RTT:
score each evidence strength, confidence
aggregate per element into delta_node[element]
derive spillover_map, borrowed_benefit, robustness_summary
Stage 5 — Derive ΔE(element) (MUST)
For each element:
propose sign from SystemDelta.delta_node[e].sign
then apply No-Guessing profiles: if profile not met → force ΔE(e)=0
Stage 6 — Robustness & Borrowed Benefit Constraints (MUST)
If flow_affecting=true:
require robustness_summary.verdict = STABLE per RTT minima
if borrowed_benefit.is_borrowed=true → cannot output TRUE_SYN (see verdict rules)
Stage 7 — Verdict Rules (MUST)
TRUE_SYN
TRUE_SYN iff:
∃e: ΔE(e)=+
and ¬∃e: ΔE(e)=−
and (if flow_affecting=true): robustness stable AND borrowed_benefit=false
FALSE_SYN
FALSE_SYN iff:
∃e: ΔE(e)=−
and harm is ROBUST per RTT (horizon+risk)
and harm is not removable via scope narrowing inside declared scope (otherwise UNKNOWN + narrow-scope plan)
UNKNOWN_SYN
Otherwise.
Stage 8 — Axis-5 secondary check (downgrade-only, MUST)
If verdict is TRUE_SYN, evaluator MUST run Axis-5:
may narrow scope OR downgrade TRUE→UNKNOWN
must not upgrade UNKNOWN→TRUE
Stage 9 — Zone mapping + Emit SVR (MUST)
Emit SVR with fields below.
## 8. SVR — Syntropy Verdict Record (Output Artifact)
SVR MUST include:
svr_id
svp_version
timestamp_utc
co_id
scope: { domain, horizon_class, boundaries }
risk_class
flow_affecting
graph_version (if flow_affecting)
syntropy_verdict
syntropy_zone
deltaE: { HUM, AI, BIO, TEC, SOC } each +|0|-
system_delta (embedded or reference)
impact_evidence_refs[]
axis5: { applied: bool, downgrade?: bool, notes[] }
limitations[]
witness_refs[]
If verdict is UNKNOWN_SYN, SVR MUST additionally include:
unknown_reasons[] (reason_code)
gap_plan_ref
## 9. Evidence Gap Loop (v0.2.2 Control Plane)
### 9.1 EvidenceGapPlan artifact (MUST on UNKNOWN_SYN)
If evaluator outputs UNKNOWN_SYN, it MUST emit EvidenceGapPlan unless stop-rule halts immediately.
EvidenceGapPlan MUST include:
gap_plan_id
co_id
svr_prev_id
timestamp_utc
unknown_reasons[]
missing_inputs[]
required_actions[] (ordered by priority)
scope_clarifications[]
stop_rule
target_horizon
risk_class
flow_affecting
reason_code enum (MUST)
MISSING_DOMAIN
MISSING_SCOPE
NOISE_BLOCK
MISSING_EVIDENCE_PROFILE
MISSING_COUPLING_EVIDENCE
INSUFFICIENT_ROBUSTNESS
BORROWED_BENEFIT_DETECTED
EVIDENCE_CONFLICT
OUT_OF_SCOPE_EVIDENCE_ONLY
RESOURCE_BLOCKED
action_type enum (MUST)
RUN_PROTOCOL_U
RUN_PROTOCOL_S_SCS
COLLECT_E_OPS
COLLECT_E_CASE
COLLECT_E_REP
COLLECT_E_FAIL
REQUEST_E_CONS
NARROW_SCOPE
SPLIT_CLAIM
ABORT_EVAL
### 9.2 Routing rules (MUST)
If reasons include any of:
MISSING_COUPLING_EVIDENCE, INSUFFICIENT_ROBUSTNESS, BORROWED_BENEFIT_DETECTED, EVIDENCE_CONFLICT
→ next action MUST target STEP 5 v1.2.2 (RUN_PROTOCOL_S_SCS and/or collect more evidence and/or scope narrowing).
If reasons include MISSING_SCOPE or MISSING_DOMAIN
→ next action MUST be rewrite via Protocol U (RUN_PROTOCOL_U, SPLIT_CLAIM, NARROW_SCOPE).
If reasons include NOISE_BLOCK
→ next action SHOULD be halt unless upstream cleaning policy exists (outside SVP).
### 9.3 Iteration cap (MUST)
Loop MUST enforce:
max_iterations (default: 3 for LOW/MED, 2 for HIGH)
each re-eval produces a new SVR referencing previous.
### 9.4 STEP 5 obligations under gap plan (MUST)
When invoked with EvidenceGapPlan, STEP 5 MUST:
execute required actions in priority order,
emit GapExecutionReport,
append new evidence refs to CO,
or emit RESOURCE_BLOCKED if actions cannot be executed.
## 10. Relationship to Step 5 Protocols U/S (normative interface)
SVP assumes STEP 5 provides:
Protocol U outputs: scope formalization, falsifiers, assumptions.
Protocol S/SCS outputs: E_sim and (optionally) E_cons.
SVP-EVAL MUST NOT run simulations itself; it only consumes evidence artifacts.
## 11. Minimal compliance checklist (engine-level)
An implementation is SVP v0.2.2 compliant iff it:
enforces Compute-Readiness gate (including coupling requirement for flow-affecting),
enforces No-Guessing profiles,
treats E_sim as evidence-only and respects SOC/BIO restrictions,
computes SystemDelta before ΔE,
applies verdict rules exactly,
applies Axis-5 downgrade-only,
emits SVR always, and EvidenceGapPlan on UNKNOWN,
supports Evidence Gap Loop with iteration caps and reports.
Если хочешь, следующим шагом я могу:
оформить SVP v0.2.2 как YAML-спеку (чтобы было проще версионировать и парсить),
и/или сгенерировать JSON Schemas для CO / Evidence / SVR / EvidenceGapPlan / GapExecutionReport.
