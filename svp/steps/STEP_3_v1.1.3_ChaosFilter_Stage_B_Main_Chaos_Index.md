# STEP 3 v1.1.3 — ChaosFilter Stage B (Main Chaos Index) + Hardened Handoff to Step 4
**Status:** Normative  
**Revision:** v1.1.3 (2026-02-14) — adds Appendix D side outputs mapping (SideRecord) without changing Step 3 structure; updates Step 4 reference to v1.1.5.
**Goal:**  
1) compute final `Chi ∈ [0..100]` for each claim (post Stage A pass),  
2) route by `Chi` thresholds,  
3) emit a deterministic **ClaimRecord (CR3)** directly consumable by **STEP 4 v1.1.4**.

**Core constraint:** Step 3 computes only chaos-related measures (variance/conflict/surface-distance/logic complexity).  
It MUST NOT assign novelty/truth/zones.

---

## 3.0 Determinism & No-Guessing (Required)

Step 3 MUST be a function ONLY of:
- `normalized_claim_text` (see 3.1)
- `policy_config_ref` (+ optional `policy_config_hash`)
- `index_snapshot_binding` (see 3.2.2)
- declared implementation versions

**No-Guessing:** if any REQUIRED policy/binding field is missing → output `BLOCKED_*` routing and do not forward to Step 4 as if OK.

---

## 3.1 Normalization (Unified with Step 4)

Step 3 MUST compute:
- `normalizer_version = policy.versions.normalizer_version`
- `normalized_claim_text = normalize(raw_claim_text, normalizer_version)`

Step 3 MUST log:
- `normalizer_version`

Normalization rules MUST be policy-controlled (not hardcoded in this document). The policy config MUST define:
- unicode normalization mode (e.g., NFKC)
- whitespace collapse
- casing mode (preserve/lower)
- url tokenization policy
- number tokenization policy
- repeated character folding policy (if any)

If `policy.versions.normalizer_version` is missing → `BLOCKED_POLICY_MISSING`.

---

## 3.2 Stage B Inputs (Required)

Stage B runs ONLY for claims that passed Stage A.

### 3.2.1 Required inputs
- Stage A handoff: `A2R.step3_handoff`:
  - `claim_id`
  - `run_context`: `{risk_class, horizon_class}`
  - `raw_input_sha256`
  - `normalized_claim_text`
  - `Chi_A`, `A_flags`
  - `policy_config_ref` (+ `policy_config_hash?`)
  - `normalizer_version`
  - `marker_pack_version`

- Runtime binding (REQUIRED for determinism when forwarding):
  - `index_snapshot_binding` (see 3.2.2)

If required inputs are missing → `BLOCKED_CONTEXT_MISSING` or `BLOCKED_POLICY_MISSING`.

### 3.2.2 Index snapshot binding (Required)

Step 3 MUST NOT read “live” B_user without snapshot binding.

`B_user_snapshot_binding` MUST include:
- `b_user_snapshot_id` (REQUIRED)
- `similarity_impl_version` (REQUIRED)
- (if embeddings used) `embedding_model_id` + `embedding_impl_version` (REQUIRED)
- (if ANN/topK used) `retrieval_impl_version` (REQUIRED)

Recommended:
- `b_user_snapshot_hash`

If any REQUIRED field missing → `BLOCKED_INDEX_UNBOUND`.

---

## 3.3 Component B1 — Surface Distance to B_user (Renamed)

**Rename:** previous `D_nov` (surface novelty) is now:

- `D_surface_distance ∈ [0..100]`

Rationale: Step 4 is the true novelty gate; Step 3 computes only surface distance w.r.t. B_user snapshot.

Compute:
1) `max_sim = max similarity(normalized_claim_text, B_user.facts ∪ B_user.boundaries)  ∈ [0..1]`
2) `D_surface_distance = clip_{0..100}(100*(1 - max_sim))`

All similarity computations MUST be bound to:
- `B_user_snapshot_binding`
- `similarity_impl_version`
- embedding model/impl versions (if used)

---

## 3.4 Component B2 — Variance via IR Stability (Versioned)

Variance measures how much the claim’s meaning shifts under deterministic paraphrase/IR variants.

### 3.4.1 Required policy fields
Step 3 MUST read variance configuration from policy:
- `policy.step3.variance.ir_builder_version`
- `policy.step3.variance.N_variants`
- `policy.step3.variance.variant_ruleset_hash`
- `policy.step3.variance.variant_max_len`
- `policy.step3.variance.feature_vector_schema`
- `policy.step3.variance.drift_metric_id`
- `policy.step3.variance.D_variance_agg`

### 3.4.2 Computation
1) Build canonical IR:
- `IR0 = IR(normalized_claim_text, ir_builder_version)`
2) Generate `N_variants` deterministic variants:
- `IRi` for `i=1..N_variants`
3) Measure semantic drift:
- `D_variance` is computed per the declared IR stability function and normalized to `[0..100]`

Step 3 MUST store:
- `ir_hash` for `IR0`
- (recommended) `ir_variants_hashes[]`

If `ir_builder_version` missing → `BLOCKED_POLICY_MISSING`.

---

## 3.5 Component B3 — Conflict vs B_user (Snapshot-bound)

Conflict measures explicit contradiction against user boundaries/facts and active conflict cases.

### 3.5.1 Required sources (all snapshot-bound)
- `B_user.boundaries`
- `B_user.facts`
- `B_user.conflicts[]` (active cases, if any)

### 3.5.2 Computation (3-tier)
Compute `D_conflict ∈ [0..100]` using the existing 3-tier model:
- **Direct contradiction** (hard boundary clash)
- **Soft conflict** (incompatible constraints)
- **Conflict-hit** (maps to existing active conflict case)

All checks MUST cite source IDs and be reproducible from `b_user_snapshot_id`.

If conflict_hit occurs, Step 3 MUST record `conflict_case_id`.

---

## 3.6 Component B4 — Logic Complexity (Diagnostic-first)

Step 3 MAY compute:
- `L_logic` (logic length/complexity proxy)
- `D_logic ∈ [0..100]` (derived logic complexity)

**Normative constraint:**
- `D_logic` MUST NOT be included in final `Chi` unless `policy.step3.logic.include_logic_in_chi=true`.
- `D_logic` MUST always be emitted as a **diagnostic** and may trigger `REWRITE_REQUIRED`.

---

## 3.7 Final Chi (Policy-bound weights)

Default (recommended) formula:
- `Chi = clip_{0..100}( 0.50*D_conflict + 0.30*D_variance + 0.20*D_surface_distance )`

However:
- weights MUST be read from policy config (not hardcoded):
- `w_conflict = policy.step3.weights.w_conflict`
- `w_variance = policy.step3.weights.w_variance`
- `w_surface_distance = policy.step3.weights.w_surface_distance`
- Step 3 MUST log the weights actually used

---

## 3.8 Routing Thresholds (Policy-bound)

Thresholds MUST be read from policy config:
- `tau_liquid = policy.step3.routing_thresholds.tau_liquid`
- `tau_human  = policy.step3.routing_thresholds.tau_human`
- `tau_border = policy.step3.routing_thresholds.tau_border`

Defaults shown in this document are illustrative only.

Routing decision:
- if `Chi ≤ tau_liquid` → `LIQUID_ROUTE`
- if `tau_liquid < Chi ≤ tau_human` → `HUMAN_ROUTE`
- if `tau_human < Chi ≤ tau_border` → `BORDERLINE_ROUTE`
- if `Chi > tau_border` → `DROP_DEFER`

Step 3 MUST log these thresholds in ClaimRecord.

---

## 3.9 Technical Overrides (Normative)

Overrides are policy-controlled but MUST map to explicit routing outcomes (not silent drops):

- If `D_conflict ≥ policy.step3.conflict.conflict_quarantine_threshold` (default 95):
  - `routing_decision = QUARANTINE_CONFLICT`
  - MUST include `conflict_case_id` or contradiction anchor IDs

- If `D_variance ≥ policy.step3.rewrite.variance_rewrite_threshold` (default 90):
  - `routing_decision = REWRITE_REQUIRED`
  - MUST emit `rewrite_request_ref`

- If Stage A already flagged `D_noise ≥ policy.step2.routing.tau_noise_drop`:
  - `routing_decision = DROP_DEFER`

---

## 3.10 Output Artifact: ClaimRecord (CR3) (Required)

Step 3 MUST emit exactly one immutable record per claim: **ClaimRecord (CR3)**.

CR3 MUST contain:

- `cr3_id`
- `claim_id` (deterministic)
- `raw_input_sha256`
- `normalized_claim_text`
- `normalizer_version`
- `policy_config_ref`, `policy_config_hash?`
- `index_snapshot_binding` (full)
- `run_context`: `{risk_class, horizon_class}`

- `stage_a`:
  - `Chi_A`
  - `A_flags`

- `stage_b`:
  - `Chi`
  - `weights_used`
  - `components`:
    - `D_conflict`
    - `D_variance`
    - `D_surface_distance`
    - `D_logic`
    - `L_logic`
  - `ir`:
    - `ir_builder_version`
    - `ir_hash`
    - `ir_variants_hashes?`

- `routing_decision`
- `rewrite_request_ref?` (if REWRITE_REQUIRED)

### 3.10.1 Step 4 handoff payload (Required if routed forward)
If `routing_decision ∈ {LIQUID_ROUTE, HUMAN_ROUTE, BORDERLINE_ROUTE}` then CR3 MUST include `step4_handoff`:

- `claim_id`
- `raw_idea` (= `normalized_claim_text` as canonical claim text)
- `run_context`: `{risk_class, horizon_class}`
- `raw_input_sha256`
- `Chi`, `D_conflict`, `D_variance`, `D_surface_distance` (audit/trace)
- `policy_config_ref` (+ `policy_config_hash?`)
- `index_snapshot_binding` (full)

**Note:** Step 4 v1.1.5 recomputes `normalized_claim_text` using `policy.versions.normalizer_version`.
Passing `raw_idea` as normalized text is allowed because the normalizer is deterministic and policy-bound.


---

## 3.11 Failure Modes (No-Guessing)

If any required policy/binding field missing:
- emit CR3 with `routing_decision = BLOCKED_POLICY_MISSING` or `BLOCKED_INDEX_UNBOUND`
- do not forward a “fake OK” handoff to Step 4

---

## 3.12 Compatibility Note
This version (v1.1.2) is required for compatibility with:
- **STEP 4 v1.1.4** (normalizer + policy binding + snapshot determinism)
- “records + hashes” reproducibility contract across the pipeline

---

# YAML Template — ClaimRecord (CR3) v1.1.2 (Normative)
claim_record_cr3_v1_1_2_template:
  cr3_id: "CR3_0001"
  class: "ClaimRecord"
  step: "STEP_3"
  version: "1.1.2"
  created_utc: "YYYY-MM-DDThh:mm:ssZ"

  claim_id: "CLM_..."
  raw_input_sha256: "sha256:..."
  normalized_claim_text: "..."
  normalizer_version: "from_policy.versions.normalizer_version"

  run_context:
    risk_class: "MED"
    horizon_class: "medium"

  policy_config_ref: "SVP-EVAL_PolicyPack_v0.1"
  policy_config_hash: null

  index_snapshot_binding:
    b_user_snapshot_id: "BUSER_..."
    b_user_snapshot_hash: null
    b_core_snapshot_id: "BCORE_..."
    b_core_snapshot_hash: null
    retrieval_impl_version: "from_policy.versions.retrieval_impl_version"
    similarity_impl_version: "from_policy.versions.similarity_impl_version"
    embedding_model_id: null
    embedding_impl_version: null
    minhash_impl_version: "from_policy.versions.minhash_impl_version"
    lsh_impl_version: "from_policy.versions.lsh_impl_version"
    simhash_impl_version: "from_policy.versions.simhash_impl_version"

  stage_a:
    Chi_A: 0
    A_flags: []

  stage_b:
    Chi: 0
    weights_used:
      w_conflict: 0.5
      w_variance: 0.3
      w_surface_distance: 0.2
    components:
      D_conflict: 0
      D_variance: 0
      D_surface_distance: 0
      D_logic: 0
      L_logic: 0
    ir:
      ir_builder_version: "ir_v1"
      ir_hash: "sha256:..."
      ir_variants_hashes: []

  routing_decision: "LIQUID_ROUTE"
  rewrite_request_ref: null

  step4_handoff:
    claim_id: "CLM_..."
    raw_idea: "..."
    run_context:
      risk_class: "MED"
      horizon_class: "medium"
    raw_input_sha256: "sha256:..."
    Chi: 0
    D_conflict: 0
    D_variance: 0
    D_surface_distance: 0
    policy_config_ref: "SVP-EVAL_PolicyPack_v0.1"
    policy_config_hash: null
    index_snapshot_binding:
      b_user_snapshot_id: "BUSER_..."
      b_core_snapshot_id: "BCORE_..."
      retrieval_impl_version: "from_policy.versions.retrieval_impl_version"
      similarity_impl_version: "from_policy.versions.similarity_impl_version"

---

## 3.13 Side outputs (Appendix D) (Required)

Step 3 MUST emit side outputs per **Appendix D — Side Outputs & Banks Contract v0.1**:
- `Appendix_D_SideOutputs_Banks_Contract_v0.1.md`
- SideRecord template example: `Appendix_D_SideRecord_Template_v0.1.yaml`

**Constraint:** this section adds an additional output channel (`side_records[]`) and mapping. It does **not** modify the CR3 artifact schema in 3.10.

### 3.13.1 When to emit SideRecord
For each processed claim, Step 3 MUST emit a SideRecord if:
- `routing_decision = DROP_DEFER` (final chaos gating fail), OR
- `routing_decision = REWRITE_REQUIRED` (normalization/rewrite required before any further semantic gates), OR
- `routing_decision` is any `BLOCKED_*` value (No-Guessing failure modes).

If `routing_decision ∈ {LIQUID_ROUTE, HUMAN_ROUTE, BORDERLINE_ROUTE}`, emitting a SideRecord is OPTIONAL (traceability and agent feedback use-case).

### 3.13.2 SideRecord mapping (normative)
For Step 3, SideRecord MUST be produced with:

- `step_id = STEP3`
- `input_ref` MUST link to `claim_id` + `raw_input_sha256`. If Step 1 provides a stable fragment pointer, it SHOULD be included.
- `normalized_claim` MUST equal `normalized_claim_text` in CR3.

**Status mapping**
- If `routing_decision = DROP_DEFER` → `status = REJECTED_CHAOS`
- If `routing_decision = REWRITE_REQUIRED` → `status = REJECTED_CHAOS`
- If `routing_decision ∈ {BLOCKED_CONTEXT_MISSING, BLOCKED_POLICY_MISSING, BLOCKED_INDEX_SNAPSHOT_MISSING}`  
  → `status = INSUFFICIENT` and `reason_codes` MUST include the blocking code.

**scores{} mapping (minimum)**
`scores` MUST include:
- Stage A: `Chi_A`
- Stage B: `Chi`
- Components: `D_conflict`, `D_variance`, `D_surface_distance`, `D_logic`, `L_logic`
- Routing thresholds used: `tau_liquid`, `tau_human`, `tau_border`
- If available: `weights_used` and a summary of triggered overrides (3.9)

**reason_codes[] mapping (minimum)**
- For `DROP_DEFER`, `reason_codes` MUST include:
  - `chaos_threshold_exceeded`
  - plus any technical overrides applied (3.9) using their override_id.
- For `REWRITE_REQUIRED`, `reason_codes` MUST include:
  - `rewrite_required`
- For `BLOCKED_*`, `reason_codes` MUST include:
  - `blocked_context_missing` OR `blocked_policy_missing` OR `blocked_index_snapshot_missing`.

**distance_to_pass mapping**
- If `DROP_DEFER` (numeric threshold):  
  `distance_to_pass = {kind:numeric, metric:Chi, threshold:tau_border, value:Chi, margin:(tau_border - Chi)}`
- If `REWRITE_REQUIRED` or `BLOCKED_*` (categorical):  
  `distance_to_pass = {kind:categorical, miss:<routing_decision>}`

**recommended_protocol mapping (default)**
- If `DROP_DEFER` → `recommended_protocol = LEARNING` (default), MAY be `CREATIVE` if the claim is intelligible but noisy.
- If `REWRITE_REQUIRED` → `recommended_protocol = LEARNING` (rewrite / simplify / extract).
- If `HUMAN_ROUTE` or `BORDERLINE_ROUTE` and SideRecord is emitted (OPTIONAL) → `recommended_protocol = PARTNER_DEV`.
- If `BLOCKED_*` → `recommended_protocol = PARTNER_DEV` (configuration / snapshot binding repair) or `LEARNING` (missing context fields).

### 3.13.3 Reopen triggers (minimum)
Step 3 SideRecords SHOULD include at least:
- `HUMAN_CHALLENGE`
- `POLICY_CHANGED` (routing thresholds or weights change)
- `TTL_EXPIRED` (OPTIONAL; only if your domain policy declares fast knowledge decay)

---

