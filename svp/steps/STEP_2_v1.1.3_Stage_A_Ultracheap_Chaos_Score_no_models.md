# STEP 2 v1.1.3 — Stage A (Ultracheap Chaos Score, no models)
**Status:** Normative  
**Revision:** v1.1.3 (2026-02-14) — adds Appendix D side outputs mapping (SideRecord) without changing Step 2 structure; updates Step 1 handoff reference to v1.1.3.
**Goal:**  
1) compute `Chi_A ∈ [0..100]` + components (variance-lite / conflict-lite / logic-lite / noise),  
2) route: cheap drop/defer vs forward to **STEP 3 Stage B**,  
3) emit an immutable **StageARecord (A2R)** for byte-level reproducibility.

**Core constraint:** Step 2 is a cheap chaos/risk filter. It MUST NOT assign truth/novelty/zones.

---

## 2.0 Determinism & No-Guessing (Required)

Everything in Step 2 MUST be a function ONLY of:
- `raw_claim_text` bytes (for hashing)
- `normalized_claim_text = normalize(raw_claim_text, normalizer_version)`
- `policy_config_ref` (+ optional `policy_config_hash`)
- `run_context`: `{risk_class, horizon_class}` (declared external inputs; required)
- declared implementation versions (tokenizer/normalizer/marker_pack)

**No-Guessing:**
- if `policy_config_ref` missing → `BLOCKED_POLICY_MISSING` (do not silently use defaults)
- if marker pack missing/unversioned → `BLOCKED_MARKER_PACK_MISSING`

---

## 2.1 Inputs (Required)

Step 2 consumes the handoff from **STEP 1 v1.1.3**.

- `step2_handoff.raw_claim_text` (string/bytes)
- `step2_handoff.claim_id` (deterministic id)
- `step2_handoff.raw_input_sha256` (RECOMMENDED; Step 2 MAY recompute for sanity)
- `step2_handoff.run_context`: `{risk_class ∈ {LOW, MED, HIGH}, horizon_class ∈ {short, medium, long}}` (REQUIRED)
- optional `step2_handoff.metadata` (ignored by Step 2 except for language hints if explicitly provided)
- `policy_config_ref` (REQUIRED), `policy_config_hash` (RECOMMENDED)

Policy MUST provide or reference (by key path):
- `policy.versions.normalizer_version` (REQUIRED)
- `policy.versions.tokenizer_version` (REQUIRED)
- `policy.marker_packs.active_marker_pack_version` + `policy.marker_packs.packs` (REQUIRED)
- `policy.step2.routing.*` thresholds (see 2.9)
- `policy.step2.weights` and `policy.step2.noise` mappings (see 2.6)
- `policy.step2.heuristics.*` (see 2.5)

---

## 2.2 Normalization (Unified with Step 3/4)

Step 2 MUST compute:
- `normalizer_version = policy.versions.normalizer_version`
- `normalized_claim_text = normalize(raw_claim_text, normalizer_version)`

Step 2 MUST log:
- `normalizer_version`

Normalization rules are policy-controlled (not hardcoded here). If `policy.versions.normalizer_version` missing → `BLOCKED_POLICY_MISSING`.

---

## 2.3 Tokenization (Policy-controlled)

Step 2 MUST compute:
- `tokenizer_version = policy.versions.tokenizer_version`
- `tokens = tokenize(normalized_claim_text, tokenizer_version)`
- `n = #tokens`

Step 2 MUST log:
- `tokenizer_version`

If `policy.versions.tokenizer_version` missing → `BLOCKED_POLICY_MISSING`.

---

## 2.4 Marker Packs (Policy-bound, Versioned)

Step 2 MUST source markers from the active marker pack referenced by policy:
- `marker_pack_version = policy.marker_packs.active_marker_pack_version` (REQUIRED)
- `marker_pack = policy.marker_packs.packs[marker_pack_version]` (REQUIRED)

`language_mode` selection MUST be deterministic:
- default: `auto`
- MAY be overridden only by explicit `metadata.language_hint` if policy permits that hint.

Step 2 MUST log:
- `marker_pack_version`
- `language_mode`

If `marker_pack_version` missing OR `packs[marker_pack_version]` missing → `BLOCKED_MARKER_PACK_MISSING`.

---

## 2.5 Feature Counts (Deterministic)

Let:
- `mV = count(tokens ∈ M_V)`
- `mA = count(tokens ∈ M_A)`
- `mL = count(tokens ∈ M_L)`

### 2.5.1 Internal contradiction (has_contra)
`has_contra` MUST be deterministic and policy-defined.

Default recommendation:
- `has_contra = 1` if `(mA > 0 AND mV > 0)` OR any token-pair matches `contradiction_pairs` from marker pack.
- else `has_contra = 0`

---

## 2.6 Noise Metrics (Strict Definitions)

Noise metrics MUST be computed from `normalized_claim_text` and `tokens`.

### 2.6.1 Symbol fraction p_sym
Let `chars` be unicode codepoints in `normalized_claim_text`.
Policy MUST define the denominator mode:
- either `total = #all chars excluding whitespace`
- or `total = #all chars including whitespace`

Compute:
- `non_alnum = count(chars where char is NOT (Letter or Number) and NOT whitespace)`
- `p_sym = non_alnum / max(1, total)`

### 2.6.2 Suspicious token fraction p_sus
Policy MUST define a versioned suspicious ruleset. Default recommended rules:

A token is suspicious if any holds:
- **R1:** token contains ≥2 script transitions among {Latin, Cyrillic, Digits}
- **R2:** token contains a run of ≥3 identical chars from set `{'_','-','*'}` (or policy-defined)
- **R3:** token length ≥ `sus_len_min` AND token has **no vowels** for the active language set
- **R4:** token matches `noise_token_patterns/spam_patterns` from marker pack

Compute:
- `p_sus = (# suspicious tokens) / max(1, n)`

### 2.6.3 len_max
- `len_max = max token length` (unicode codepoints)

### 2.6.4 url_count
If the normalizer emits `<url>`:
- `url_count = count(token == "<url>")`

### 2.6.5 repeat_punct_flag
Policy defines `repeat_punct_run` (default 4).
- `repeat_punct_flag = true` if any punctuation run length ≥ `repeat_punct_run`

---

## 2.7 Component Formulas

All clip operators MUST be executable:
- `clip_{0..100}(x) = min(100, max(0, x))`

### 2.7.1 A1 Variance-lite
`has_dangling_deictics` MUST be deterministic and policy-defined.

Default surrogate:
- deictic token present AND no “content token” present  
(content token heuristic is policy-controlled; default: length ≥ 4, not marker, not `<num>`, not `<url>`)

Compute:
- `D_A_var = clip_{0..100}( 20*min(3,mV) + 15*I[mV>=4] + 15*I[has_dangling_deictics] )`

### 2.7.2 A2 Conflict-lite
- `D_A_conf = clip_{0..100}( 70*has_contra + 10*min(3,mA) )`

### 2.7.3 A3 Logic-overload-lite
`has_nested_conditions` MUST be deterministic and policy-defined.

Default surrogate:
- count("если") ≥ 2 OR ("если" AND "при условии") both present (token forms policy-defined)

Compute:
- `D_A_logic = clip_{0..100}( 10*min(6,mL) + 20*I[mL>=7] + 15*I[has_nested_conditions] )`

### 2.7.4 Noise score D_noise (Numeric, Policy-bound)
Noise is computed as:

`D_noise = clip_{0..100}(
    w_sym  * f_sym(p_sym)
  + w_sus  * f_sus(p_sus)
  + w_len  * f_len(len_max)
  + w_url  * I[url_count > url_max)
  + w_punct* I[repeat_punct_flag]
)`

Where:
- weights `w_*` MUST come from policy config
- `f_sym`, `f_sus`, `f_len` MUST be declared deterministic mappings (piecewise linear or lookup) in policy
- `url_max` and `repeat_punct_run` come from policy

---

## 2.8 Aggregate Chi_A (Policy-bound Weights)

Default recommended aggregator:
- `Chi_A = clip_{0..100}( a_var*D_A_var + a_conf*D_A_conf + a_logic*D_A_logic + a_noise*D_noise )`

But:
- weights `a_*` MUST be read from policy config
- Step 2 MUST log the weights used

---

## 2.9 Routing (Policy-bound)

Policy MUST define:
- `tau_stageA_drop = policy.step2.routing.tau_stageA_drop`
- `tau_noise_drop  = policy.step2.routing.tau_noise_drop`

Routing decision:
- if `BLOCKED_*` → `BLOCKED_*`
- else if `D_noise ≥ tau_noise_drop` → `DROP_DEFER`
- else if `Chi_A > tau_stageA_drop` → `DROP_DEFER`
- else → `FORWARD_TO_STEP3`

---

## 2.10 Output Artifact — StageARecord (A2R) (Required)

Step 2 MUST emit exactly one immutable **StageARecord** per claim.

### 2.10.1 StageARecord fields (Normative)
A2R MUST include:
- `a2r_id`
- `claim_id` (deterministic; SHOULD match Step 1)
- `run_context`: `{risk_class, horizon_class}`
- `raw_input_sha256`
- `normalized_claim_text`
- `normalizer_version`
- `tokenizer_version`
- `policy_config_ref`, `policy_config_hash?`
- `marker_pack_version`
- `language_mode`

- `counts`: `{n, mV, mA, mL, has_contra}`
- `noise_inputs`: `{p_sym, p_sus, len_max, url_count, repeat_punct_flag}`

- `components`: `{D_A_var, D_A_conf, D_A_logic, D_noise}`
- `weights_used`: `{a_var, a_conf, a_logic, a_noise, w_sym, w_sus, w_len, w_url, w_punct, url_max, repeat_punct_run}`
- `Chi_A`

- `A_flags[]`:
  each flag MUST include `{flag_id, triggered, contribution, note}`

- `routing_decision ∈ {FORWARD_TO_STEP3, DROP_DEFER, BLOCKED_CONTEXT_MISSING, BLOCKED_POLICY_MISSING, BLOCKED_MARKER_PACK_MISSING}`
- `created_utc`

### 2.10.2 Step 3 handoff payload (Required if forwarded)
If `routing_decision = FORWARD_TO_STEP3`, A2R MUST include `step3_handoff`:
- `claim_id`
- `run_context`: `{risk_class, horizon_class}`
- `raw_input_sha256`
- `normalized_claim_text`
- `Chi_A`, `A_flags`
- `policy_config_ref` (+ `policy_config_hash?`)
- `normalizer_version`
- `marker_pack_version`

---

## 2.11 Failure Modes (No-Guessing)

If required run-context fields are missing:
- emit A2R with `routing_decision = BLOCKED_CONTEXT_MISSING`
- do not forward

If required policy/marker fields are missing:
- emit A2R with `routing_decision = BLOCKED_POLICY_MISSING` or `BLOCKED_MARKER_PACK_MISSING`
- do not forward

---

## 2.12 Compatibility Note
This version (v1.1.2) is required for compatibility with:
- **STEP 1 v1.1.3** (RunContext + deterministic claim_id handoff)
- **STEP 3 v1.1.2** (handoff consumes A2R.step3_handoff)
- **STEP 4 v1.1.4** (end-to-end determinism and auditability)

---

# YAML Template — StageARecord (A2R) v1.1.2 (Normative)
stage_a_record_a2r_v1_1_2_template:
  a2r_id: "A2R_0001"
  class: "StageARecord"
  step: "STEP_2"
  version: "1.1.2"
  created_utc: "YYYY-MM-DDThh:mm:ssZ"

  claim_id: "CLM_..."
  raw_input_sha256: "sha256:..."

  run_context:
    risk_class: "MED"
    horizon_class: "medium"

  normalized_claim_text: "..."
  normalizer_version: "norm_v1"
  tokenizer_version: "tok_v1"

  policy_binding:
    policy_config_ref: "POLICY_..."
    policy_config_hash: "sha256:..."

  marker_binding:
    marker_pack_version: "MARKERS_RU_v1"
    language_mode: "auto"

  counts:
    n: 42
    mV: 1
    mA: 0
    mL: 2
    has_contra: 0

  noise_inputs:
    p_sym: 0.03
    p_sus: 0.00
    len_max: 12
    url_count: 0
    repeat_punct_flag: false

  components:
    D_A_var: 20
    D_A_conf: 0
    D_A_logic: 20
    D_noise: 5

  weights_used:
    a_var: 0.25
    a_conf: 0.35
    a_logic: 0.20
    a_noise: 0.20
    w_sym: 1.0
    w_sus: 1.0
    w_len: 1.0
    w_url: 15.0
    w_punct: 10.0
    url_max: 1
    repeat_punct_run: 4

  Chi_A: 14

  A_flags:
    - flag_id: "FLAG_MV_PRESENT"
      triggered: true
      contribution: 20
      note: "mV=1"
    - flag_id: "FLAG_NOISE_LOW"
      triggered: true
      contribution: 0
      note: "D_noise=5"

  routing_decision: "FORWARD_TO_STEP3"

  step3_handoff:
    claim_id: "CLM_..."
    run_context_ref: "A2R_0001.run_context"
    raw_input_sha256: "sha256:..."
    normalized_claim_text: "..."
    Chi_A: 14
    A_flags_ref: "A2R_0001"
    policy_config_ref: "POLICY_..."
    policy_config_hash: "sha256:..."
    normalizer_version: "norm_v1"
    marker_pack_version: "MARKERS_RU_v1"

---

## 2.13 Side outputs (Appendix D) (Required)

Step 2 MUST emit side outputs per **Appendix D — Side Outputs & Banks Contract v0.1**:
- `Appendix_D_SideOutputs_Banks_Contract_v0.1.md`
- SideRecord template example: `Appendix_D_SideRecord_Template_v0.1.yaml`

**Constraint:** this section adds an additional output channel (`side_records[]`) and mapping. It does **not** modify the A2R artifact schema defined in 2.10.

### 2.13.1 When to emit SideRecord
For each processed claim, Step 2 MUST emit a SideRecord if:
- `routing_decision = DROP_DEFER` (cheap chaos gating fails), OR
- `routing_decision` is any `BLOCKED_*` value (No-Guessing failure).

If `routing_decision = FORWARD_TO_STEP3`, emitting a SideRecord is OPTIONAL (traceability use-case).

### 2.13.2 SideRecord mapping (normative)
For Step 2, SideRecord MUST be produced with:

- `step_id = STEP2`
- `input_ref` MUST link to `claim_id` + the Step 1 intake fragment.
- `normalized_claim` MUST equal `normalized_claim_text` produced in 2.2.

**Status mapping**
- If `routing_decision = DROP_DEFER` → `status = REJECTED_CHAOS`
- If `routing_decision ∈ {BLOCKED_CONTEXT_MISSING, BLOCKED_POLICY_MISSING, BLOCKED_MARKER_PACK_MISSING}`  
  → `status = INSUFFICIENT` and `reason_codes` MUST include the blocking code.

**scores{} mapping (minimum)**
`scores` MUST include:
- `Chi_A`
- `D_A_var`, `D_A_conf`, `D_A_logic`, `D_noise`
- `p_sym`, `p_sus`, `len_max`, `url_count`, `repeat_punct_flag`
- `n`, `mV`, `mA`, `mL`, `has_contra`

**reason_codes[] mapping**
- For `REJECTED_CHAOS`, `reason_codes` MUST include:
  - `chaos_threshold_exceeded`
  - plus any triggered `A_flags[].flag_id` where `triggered=true` and `contribution>0`.
- For `BLOCKED_*`, `reason_codes` MUST include:
  - `blocked_context_missing` OR `blocked_policy_missing` OR `blocked_marker_pack_missing`.

**distance_to_pass mapping**
- If `DROP_DEFER` (numeric threshold):  
  `distance_to_pass = {kind:numeric, metric:Chi_A, threshold:<policy.step2.routing.drop_if_ge>, value:Chi_A, margin:(threshold - Chi_A)}`
- If `BLOCKED_*` (categorical):  
  `distance_to_pass = {kind:categorical, miss:<routing_decision>}`

**recommended_protocol mapping (default)**
- If `REJECTED_CHAOS` → `recommended_protocol = LEARNING` (default), MAY be `CREATIVE` if the claim is intelligible but noisy.
- If `BLOCKED_CONTEXT_MISSING` → `recommended_protocol = LEARNING` (request missing RunContext fields).
- If `BLOCKED_POLICY_MISSING` or `BLOCKED_MARKER_PACK_MISSING` → `recommended_protocol = PARTNER_DEV` (configuration/pack repair).

### 2.13.3 Reopen triggers (minimum)
Step 2 SideRecords SHOULD include at least:
- `HUMAN_CHALLENGE`
- `POLICY_CHANGED` (if policy or thresholds change)
- `TTL_EXPIRED` (OPTIONAL; only if your domain policy declares fast knowledge decay)

---

