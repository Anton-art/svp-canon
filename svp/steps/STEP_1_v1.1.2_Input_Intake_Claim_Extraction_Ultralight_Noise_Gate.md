# STEP 1 v1.1.2 — Input Intake + Claim Extraction + Ultralight Noise Gate
**Status:** Normative (startup defaults; calibration expected)  
**Goal:**  
1) accept raw messages and deterministically extract claim units,  
2) compute an ultralight noise/format-risk gate,  
3) emit a **Step1Record (S1R)** and route to **STEP 2**.

**Calibration note:** all thresholds/weights in this step are **STARTUP DEFAULTS** expected to be revised empirically, but during any run they MUST be **policy-bound and versioned** for reproducibility.
**Revision:** v1.1.2 (2026-02-13) — added RunContext (risk/horizon) contract; propagated into S1R + Step2 handoff; added BLOCKED_CONTEXT_MISSING.

---

## 1.0 Input Contract (RawMessage)

Step 1 MUST accept an input object `RawMessage`:

- `message_id` (required)
- `source_type ∈ {human, agent, system, web}` (required)
- `raw_text` (required)
- `received_utc` (required)
- `language_hint` (optional)
- `metadata` (optional; if present MUST be versioned: `metadata_schema_version`)

Step 1 MUST compute:
- `raw_input_sha256 = sha256(raw_text bytes)`

Step 1 MUST also accept a `RunContext` object (required for downstream steps):

- `risk_class ∈ {LOW, MED, HIGH}` (REQUIRED)
- `horizon_class ∈ {short, medium, long}` (REQUIRED)
- `stakes_profile_ref` (optional; policy-defined identifiers only)
- `run_id` (optional; for traceability across steps)

**No-Guessing:** if `risk_class` or `horizon_class` is missing → `BLOCKED_CONTEXT_MISSING`.

---

## 1.1 Policy Binding (Required)

Step 1 MUST be bound to policy config:
- `policy_config_ref` (REQUIRED)
- `policy_config_hash` (RECOMMENDED)

Policy MUST supply:
- `normalizer_version`
- `tokenizer_version`
- `claim_splitter_version`
- `s1_limits` (min/max chars/tokens)
- `s1_noise_thresholds` (see 1.4)


Run classification (`risk_class`, `horizon_class`) MUST come from `RunContext` and MUST NOT be inferred from the claim text in Step 1.
**No-Guessing:** if `policy_config_ref` is missing → `BLOCKED_POLICY_MISSING`.

---

## 1.2 Normalization & Tokenization Hooks (Unified)

Step 1 MUST NOT invent its own normalization rules.

It MUST compute:
- `normalized_text = normalize(raw_text, normalizer_version)`
- `tokens = tokenize(normalized_text, tokenizer_version)`

And MUST log:
- `normalizer_version`
- `tokenizer_version`

If either is missing → `BLOCKED_POLICY_MISSING`.

---

## 1.3 Claim Extraction (Deterministic)

Step 1 MUST deterministically split `normalized_text` into claim units:
- `claims = split_into_claims(normalized_text, claim_splitter_version)`

`claim_splitter_version` MUST be policy-defined.

### 1.3.1 Startup default splitter rules (policy-controlled)
- Split on sentence boundary punctuation: `. ? ! ;`
- Split on list separators: newline bullets (`-`, `*`) and numbered items (`1)`, `1.`)
- Preserve quoted blocks as separate claims if quote prefix detected (e.g., `>` or quotation markers)
- Drop empty claims after trimming

For each claim, Step 1 MUST emit:
- `claim_index` (0..)
- `claim_text` (or a pointer to stored text)
- `claim_sha256`
- `claim_len_chars`
- `claim_len_tokens`

If `claim_splitter_version` missing → `BLOCKED_POLICY_MISSING`.

---

## 1.4 Ultralight Noise Gate (Format-risk prefilter)

**Important:** Step 1 is NOT the full Stage A scorer (that is Step 2).  
Step 1 only blocks obvious garbage and flags injection-like / format-risk patterns cheaply.

Step 1 MUST compute primitives deterministically (definitions are shared with Step 2 v1.1.1):
- `n = #tokens`
- `p_sym` (symbol fraction)
- `p_sus` (suspicious token fraction)
- `len_max` (max token length)
- `url_count` (count of `<url>` tokens if normalizer emits them)
- `repeat_punct_flag` (punctuation run ≥ `policy.repeat_punct_run`)

### 1.4.1 Startup default thresholds (policy-bound)
Policy SHOULD provide these defaults (can be calibrated later):

**Limits**
- `s1_min_chars = 3`
- `s1_min_tokens = 2`
- `s1_max_chars = 20000`
- `s1_max_tokens = 4000`

**Block/Defer thresholds**
- `s1_block_p_sym = 0.35`
- `s1_block_p_sus = 0.25`
- `s1_block_len_max = 80`
- `s1_block_url_count = 5`
- `s1_block_repeat_punct = true`

### 1.4.2 Decision rules (Normative)
Routing decision is computed in this order:

1) If `raw_text` too short (`chars < s1_min_chars` OR `tokens < s1_min_tokens`) → `DROP_TOO_SHORT`
2) If `raw_text` too long  (`chars > s1_max_chars` OR `tokens > s1_max_tokens`) → `DEFER_TOO_LONG`
3) If `p_sym ≥ s1_block_p_sym` → `DROP_NOISE`
4) Else if `p_sus ≥ s1_block_p_sus` → `DROP_SUSPICIOUS`
5) Else if `len_max ≥ s1_block_len_max` → `DEFER_FORMAT_RISK`
6) Else if `url_count > s1_block_url_count` → `DROP_SPAM_URL`
7) Else if (`repeat_punct_flag` AND `s1_block_repeat_punct=true`) → `DEFER_FORMAT_RISK`
8) Else → `PASS_TO_STEP2`

### 1.4.3 Semantics of outcomes (Normative)
- `DROP_*` = do not schedule further compute
- `DEFER_*` = store to queue/quarantine for later inspection/retry
- `PASS_TO_STEP2` = forward claims to Step 2

---

## 1.5 Output Artifact — Step1Record (S1R)

Step 1 MUST emit exactly one immutable record per input message: **Step1Record (S1R)**.

### 1.5.1 S1R required fields
S1R MUST include:
- `s1r_id`
- `message_id`
- `raw_input_sha256`
- `source_type`
- `received_utc`
- `run_context`:
  - `risk_class`
  - `horizon_class`
  - `stakes_profile_ref?`
  - `run_id?`
- `policy_config_ref`, `policy_config_hash?`
- `normalizer_version`, `tokenizer_version`, `claim_splitter_version`
- `normalized_text_sha256`
- `claims[]`:
  - `claim_index`
  - `claim_sha256`
  - `claim_len_chars`
  - `claim_len_tokens`
  - `claim_text_ref` (pointer or hash-based ref)
- `noise_primitives`: `{p_sym, p_sus, len_max, url_count, repeat_punct_flag}`
- `thresholds_used`: all `s1_*` thresholds/limits
- `routing_decision ∈ {PASS_TO_STEP2, DROP_TOO_SHORT, DROP_NOISE, DROP_SUSPICIOUS, DROP_SPAM_URL,
                       DEFER_TOO_LONG, DEFER_FORMAT_RISK, BLOCKED_POLICY_MISSING, BLOCKED_CONTEXT_MISSING}`
- `created_utc`

### 1.5.2 Step 2 handoff (Required if passed)
If `routing_decision = PASS_TO_STEP2`, S1R MUST include `step2_handoff`:
- `risk_class`
- `horizon_class`
- `stakes_profile_ref?`
- `run_id?`
- `message_id`
- `claims[]` (claim references)
- `policy_config_ref` (+ `policy_config_hash?`)
- `normalizer_version`, `tokenizer_version`

---

## 1.6 Failure Modes (No-Guessing)

If any required policy binding or required version identifier is missing:
- emit `routing_decision = BLOCKED_POLICY_MISSING`
- do not pass to Step 2

---

## 1.7 Compatibility

This step aligns primitives and versioning with:
- Step 2 v1.1.1 (noise primitives + policy binding)
- Step 3 v1.1.1 and Step 4 v1.1.1 (shared normalizer + traceability)

---

# YAML Template — Step1Record (S1R) v1.1.2 (Normative)
step1_record_s1r_v1_1_2_template:
  s1r_id: "S1R_0001"
  class: "Step1Record"
  step: "STEP_1"
  version: "1.1.2"
  created_utc: "YYYY-MM-DDThh:mm:ssZ"

  message_id: "MSG_..."
  source_type: "human"              # human|agent|system|web
  received_utc: "YYYY-MM-DDThh:mm:ssZ"

  run_context:
    risk_class: "MED"              # LOW|MED|HIGH
    horizon_class: "short"         # short|medium|long
    stakes_profile_ref: null         # optional
    run_id: null                     # optional

  raw_input_sha256: "sha256:..."
  normalized_text_sha256: "sha256:..."

  policy_binding:
    policy_config_ref: "POLICY_..."
    policy_config_hash: "sha256:..."

  versions:
    normalizer_version: "norm_v1"
    tokenizer_version: "tok_v1"
    claim_splitter_version: "split_v0_1"

  claims:
    - claim_index: 0
      claim_sha256: "sha256:..."
      claim_len_chars: 84
      claim_len_tokens: 14
      claim_text_ref: "PTR:store://claims/MSG_.../0"
    - claim_index: 1
      claim_sha256: "sha256:..."
      claim_len_chars: 62
      claim_len_tokens: 10
      claim_text_ref: "PTR:store://claims/MSG_.../1"

  noise_primitives:
    p_sym: 0.05
    p_sus: 0.00
    len_max: 12
    url_count: 0
    repeat_punct_flag: false

  thresholds_used:
    s1_min_chars: 3
    s1_min_tokens: 2
    s1_max_chars: 20000
    s1_max_tokens: 4000
    s1_block_p_sym: 0.35
    s1_block_p_sus: 0.25
    s1_block_len_max: 80
    s1_block_url_count: 5
    s1_block_repeat_punct: true
    repeat_punct_run: 4

  routing_decision: "PASS_TO_STEP2"  # or DROP_*/DEFER_*/BLOCKED_*

  step2_handoff:
    message_id: "MSG_..."
    claims_ref: "PTR:store://claims/MSG_.../*"
    policy_config_ref: "POLICY_..."
    policy_config_hash: "sha256:..."
    risk_class: "MED"
    horizon_class: "short"
    stakes_profile_ref: null
    run_id: null
    normalizer_version: "norm_v1"
    tokenizer_version: "tok_v1"
