# STEP 4 v1.1.5 — Novelty & Connectedness Gate
**Status:** Normative  
**Purpose:** classify input as **KNOWN / NEAR_DUP / NOVEL_CONNECTED / NOVEL_ORPHAN** and route accordingly.  
**Revision:** v1.1.5 (2026-02-14) — adds Appendix D side outputs mapping (SideRecord) without changing Step 4 structure.
**Core constraint:** Step 4 is a **gate + routing step**, not a truth step.

---

## 4.1 Inputs (Required)

### 4.1.1 Claim inputs
- `raw_idea` (text)
- `metadata` (optional): tags, domain hints **only if explicitly provided**, timestamps, author id

### 4.1.1.1 Personal agent metadata channel (Normative)
If present, `metadata` SHOULD be emitted by the user's personal AI-agent and MUST be treated as:
- explicitly provided (not inferred)
- machine-readable
- versioned (`metadata_schema_version`)

Step 4 MUST NOT infer tags/domain from prose content. Tags are accepted only if explicit in metadata.

### 4.1.2 Knowledge bases (required)
- `B_user` (user memory / user base)
- `B_core` (core/base knowledge)

### 4.1.3 Policy + bindings (required)
- `policy_config_ref` **(REQUIRED)**
- `policy_config_hash` (RECOMMENDED)

### 4.1.4 Index snapshot binding (required)
`index_snapshot_binding` MUST include:
- `b_user_snapshot_id`
- `b_core_snapshot_id`
- `retrieval_impl_version`

Recommended (auditability / reproducibility):
- `b_user_snapshot_hash`
- `b_core_snapshot_hash`

If embeddings are used:
- `embedding_model_id`
- `embedding_impl_version`
- `similarity_impl_version`

If `policy.step4.near_dup.enabled=true`, the binding SHOULD also include (audit):
- `shingler_version`
- `minhash_impl_version`
- `lsh_impl_version`
- `simhash_impl_version`

### 4.1.5 Risk input (required for orphan routing)
- `risk_class ∈ {LOW, MED, HIGH}`

---

## 4.2 Metrics (Executable)

Step 4 computes connectedness between `raw_idea` and nearest candidates in `B_user` and `B_core`.

### 4.2.1 Lexical connectedness
For candidate `c`:

- `C_lex = Jaccard(ngrams(raw_idea), ngrams(c))  ∈ [0..1]`  (exact computation on the final candidate set)

Alternative lexical functions MAY be used only if:
- deterministic
- versioned (`lex_impl_version`)
- outputs normalized to `[0..1]`

### 4.2.2 Semantic connectedness (embedding channel)
If embeddings are used:

- `C_sem = cosine_sim(emb(raw_idea), emb(c)) ∈ [-1..1]`
- Normalize:
  - `C_sem01 = clip_{0..1}((C_sem + 1)/2)`

If embeddings are not used, a deterministic semantic analogue MAY be used only if:
- explicitly specified
- versioned
- normalized to `[0..1]`

### 4.2.3 Combined connectedness
Step 4 MUST be policy-bound on whether semantic scoring is used:
- `use_semantic = policy.step4.scoring.use_semantic` (bool)

Let `α = policy.step4.scoring.alpha`.

If `use_semantic=true`:
- `C_connect = clip_{0..1}( α*C_lex + (1-α)*C_sem01 )`

If `use_semantic=false` (semantic channel disabled):
- Step 4 MUST set `α = 1.0` and compute `C_connect = C_lex`.
- `C_sem01` MUST NOT be computed.

**No-Guessing:** if `use_semantic=false` but `α != 1.0` → `BLOCKED_POLICY_MISSING`.

### 4.2.4 Claim text normalization (Normative)
Before retrieval and `co_id` derivation, Step 4 MUST compute:

- `normalizer_version = policy.versions.normalizer_version`
- `normalized_claim_text = normalize(raw_idea, normalizer_version)`

Step 4 MUST log:
- `normalizer_version`

Normalization MUST be deterministic.

Recommended defaults (policy-controlled via `normalizer_version`):
- unicode: NFKC
- whitespace collapse
- no stopword removal (unless policy explicitly enables)
- casing: policy-defined (default: preserve)

**No-Guessing:** if `policy.versions.normalizer_version` is missing → `BLOCKED_POLICY_MISSING`.

### 4.2.5 Near-duplicate fingerprints (MinHash/LSH + SimHash) (Optional, policy-bound)
This block is **independent** of semantic scoring and exists to accelerate and harden NEAR_DUP routing.

Enabled iff `policy.step4.near_dup.enabled=true`.

**MinHash/LSH (lexical candidate generation):**
- Build shingles from `normalized_claim_text` using `shingle_k = policy.step4.near_dup.shingle_k`.
- Compute MinHash signature of length `minhash_k = policy.step4.near_dup.minhash_k`.
- Use LSH with:
  - `lsh_bands = policy.step4.near_dup.lsh_bands`
  - `lsh_rows  = policy.step4.near_dup.lsh_rows`
  - Constraint: `lsh_bands * lsh_rows == minhash_k` else `BLOCKED_POLICY_MISSING`.
- LSH yields candidate ids from each base: `cands_lsh_user`, `cands_lsh_core`.
- For each LSH candidate, Step 4 MAY compute an **estimated** Jaccard:
  - `J_est = match_fraction(minhash_sig_claim, minhash_sig_candidate)`
  - and flag `minhash_dup = (J_est ≥ tau_dup_jaccard_est)` where:
    - `tau_dup_jaccard_est = policy.step4.near_dup.tau_dup_jaccard_est`.

**SimHash (near-dup confirmation):**
- `simhash_bits = policy.step4.near_dup.simhash_bits` (default 64)

- Compute `simhash_u64` over deterministic tokens of `normalized_claim_text` (implementation MUST be versioned).
- For each candidate with known simhash, compute Hamming distance:
  - `H = hamming(simhash_u64_claim, simhash_u64_candidate)`
  - flag `simhash_dup = (H ≤ tau_dup_simhash_hamming)` where:
    - `tau_dup_simhash_hamming = policy.step4.near_dup.tau_dup_simhash_hamming`.

**Near-dup signal:**
- `dup_signal(c) = minhash_dup OR simhash_dup`

**No-Guessing:** If near-dup is enabled but any required key is missing → `BLOCKED_POLICY_MISSING`.

---

## 4.3 Policy inputs & key paths (Required)

Step 4 MUST read thresholds and switches from the policy pack.

**Core keys:**
- `policy.step4.scoring.use_semantic` (bool)
- `policy.step4.scoring.alpha` (float)
- `policy.versions.normalizer_version` (string)
- `policy.versions.retrieval_impl_version` (string)
- `policy.versions.similarity_impl_version` (string)

**Connectedness thresholds:**
- `policy.step4.thresholds.tau_known` (float)
- `policy.step4.thresholds.tau_near` (float)
- `policy.step4.thresholds.tau_orphan` (float)

**Retrieval keys:**
- `policy.step4.retrieval.K_default` (int)

**Near-dup keys (only if enabled):**
- `policy.step4.near_dup.enabled` (bool)
- `policy.step4.near_dup.shingle_k` (int)
- `policy.step4.near_dup.minhash_k` (int)
- `policy.step4.near_dup.lsh_bands` (int)
- `policy.step4.near_dup.lsh_rows` (int)
- `policy.step4.near_dup.simhash_bits` (int)
- `policy.step4.near_dup.tau_dup_jaccard_est` (float)
- `policy.step4.near_dup.tau_dup_simhash_hamming` (int)
- `policy.versions.minhash_impl_version` (string)
- `policy.versions.lsh_impl_version` (string)
- `policy.versions.simhash_impl_version` (string)

**Orphan-consensus binding (only if NOVEL_ORPHAN is produced):**
- `policy.orphan_consensus.enabled` (bool)
- `policy.orphan_consensus.protocol_ref` (string)
- `policy.orphan_consensus.quorum.min_humans` (int)
- `policy.orphan_consensus.quorum.min_agents` (int)
- `policy.orphan_consensus.required_fields[]` (list)

**No-Guessing:** missing any REQUIRED key → `BLOCKED_POLICY_MISSING`.

## 4.4 Retrieval (Deterministic)

### 4.4.1 Binding validation (No-Guessing)
If any REQUIRED field in `index_snapshot_binding` is missing:
- emit `BLOCKED_INDEX_UNBOUND`
- do **not** perform heuristic retrieval

### 4.4.2 Retrieval procedure
Step 4 performs **candidate generation** then **exact scoring**.

**A) Primary retrieval (deterministic topK):**
- `top_neighbors_user = topK(B_user, normalized_claim_text, K)`
- `top_neighbors_core = topK(B_core, normalized_claim_text, K)`

**B) Near-dup candidate expansion (optional):**
If `policy.step4.near_dup.enabled=true`, Step 4 MUST additionally query LSH buckets using MinHash:
- `cands_lsh_user = LSH_lookup(B_user, minhash_sig_claim)`
- `cands_lsh_core = LSH_lookup(B_core, minhash_sig_claim)`

**C) Final candidate set:**
- `CAND = dedup(top_neighbors_user ∪ top_neighbors_core ∪ cands_lsh_user ∪ cands_lsh_core)`

**D) Exact scoring on CAND:**
For each candidate `n ∈ CAND`, compute:
- `C_lex(n)`, `C_sem01(n)` (if enabled), `C_connect(n)`
- if near-dup enabled and fingerprints available: `J_est(n)`, `H(n)` and `dup_signal(n)`

Step 4 MUST log:
- `K` used
- filters applied (if any)
- `CAND_size`, and per-channel counts (topK vs LSH)
- per-base neighbor lists with scores (topK lists remain explicit)
- best-match fingerprints (if enabled)

### 4.4.3 K defaults + retrieval params (Normative)
Step 4 MUST read retrieval parameters from policy config:
- `K_default` **(REQUIRED)**
- `retrieval_filters` (optional)
- `language_mode` (optional)
- `max_candidate_len / min_candidate_len` (optional)

If `K` not explicitly provided, Step 4 MUST use `K_default`.  
If `K_default` missing → `BLOCKED_POLICY_MISSING`.

### 4.4.4 Neighbor id type (Normative)
Each neighbor MUST expose an explicit id type:

- `neighbor_id_type ∈ {knowledge_item_id, co_id, assertion_id}`

Step 4 MUST record `neighbor_id_type` in KGR.  
If unknown → `BLOCKED_INDEX_UNBOUND` (No-Guessing).

---

## 4.5 Classification (Deterministic)

Let:
- `m = max(C_connect)` over all candidates from both bases.
- `best = argmax(C_connect)` with deterministic tie-break (4.5.1).

Baseline classification (connectedness thresholds):
- if `m ≥ τ_known` → `class = KNOWN`
- else if `m ≥ τ_near` → `class = NEAR_DUP`
- else if `m ≥ τ_orphan` → `class = NOVEL_CONNECTED`
- else → `class = NOVEL_ORPHAN`

Near-dup override (policy-bound; optional):
If `policy.step4.near_dup.enabled=true` and `dup_signal(best)=true`, then:
- if `C_lex(best) ≥ τ_known` → force `class = KNOWN`
- else if `C_lex(best) ≥ τ_near` → force `class = NEAR_DUP`
- else: keep baseline classification (do not force).

Step 4 MUST record:
- `best_match_id`
- `best_match_base ∈ {B_user, B_core}`
- `best_match_scores = {C_lex, C_sem01, C_connect}`
- if near-dup enabled: `best_match_dup = {J_est?, H?, dup_signal}`

### 4.5.1 Tie-breaker rule (Normative)
If multiple candidates share identical maximal `C_connect` (within equality under implementation precision),
Step 4 MUST select best match deterministically by:

1) higher `C_connect`  
2) higher `C_sem01`  
3) higher `C_lex`  
4) lexicographically smallest candidate id (as string)

Step 4 MUST record `tie_break_applied: true/false`.

---

## 4.6 Outputs (Required Artifact Contract)

Step 4 MUST emit:

### 4.6.1 KnownnessGateRecord (KGR)
A deterministic audit record for the gate decision.

KGR MUST include:
- `kgr_id`
- `co_id` (see 4.6.2)
- `class ∈ {KNOWN, NEAR_DUP, NOVEL_CONNECTED, NOVEL_ORPHAN, BLOCKED_POLICY_MISSING, BLOCKED_INDEX_UNBOUND}`
- `policy_config_ref`, `policy_config_hash?`
- `normalizer_version`
- `index_snapshot_binding` (full)
- `neighbor_id_type`
- `K`, filters
- `top_neighbors_user[]` and `top_neighbors_core[]` (id + scores)
- `candidate_set_summary = {cand_size, topk_user_count, topk_core_count, lsh_user_count?, lsh_core_count?}`
- `best_match_*` fields
- `tie_break_applied`
- `gating_time_utc`

If `policy.step4.near_dup.enabled=true`, KGR MUST also include:
- `near_dup_policy = {shingle_k, minhash_k, simhash_bits, tau_dup_jaccard_est, tau_dup_simhash_hamming, lsh_bands, lsh_rows}`
- `fingerprints_claim = {minhash_sig?, simhash_u64?}`
- `lsh_audit = {buckets_hit_user?, buckets_hit_core?}`
- `best_match_dup = {J_est?, H?, dup_signal}`

### 4.6.2 Deterministic co_id rule
`co_id` MUST be deterministic:

- If scope/terms already exist:
  - `co_id = hash(normalized_claim_text + scope_ref + terms_ref + policy_config_ref) truncated`
- If scope/terms not yet exist:
  - `co_id = hash(normalized_claim_text + policy_config_ref) truncated`
  - `co_id_status = "provisional"`

---

## 4.7 Routing Rules (Normative)

### 4.7.1 KNOWN
KNOWN means “high connectedness to an existing entry”.

#### 4.7.1.1 Fully resolved criterion (Normative)
Step 4 MAY treat KNOWN as “fully resolved within scope” ONLY if all hold:
- `has_canonical_entry = true`
- `no_scope_diff = true`
- `no_constraint_diff = true`

If any are unknown/false → route as NEAR_DUP (diff/ProofPlan required).

If fully resolved:
- route to Step 6 with `KGR` + optional evidence refs

Else:
- emit `DiffStub` and route to Step 5

### 4.7.2 NEAR_DUP
NEAR_DUP means “very close to existing entry but may differ in scope/constraints”.

Step 4 MUST emit a `DiffStub`:
- `diff_type ∈ {SCOPE_CHANGE, CONSTRAINT_CHANGE, WORDING_ONLY, UNKNOWN}`
- `base_ref` (anchor id)
- `diff_notes`

#### 4.7.2.1 DiffStub minimal computable fields (Normative)
DiffStub MUST include:
- `neighbor_id_type`
- `anchor_id`
- `detected_differences: {scope: bool|unknown, constraints: bool|unknown, wording: bool|unknown}`

If all differences are unknown → `diff_type MUST be UNKNOWN` (No-Guessing).

Routing:
- if `diff_type = WORDING_ONLY` → may short-circuit to KNOWN handling
- else → route to Step 5 (ProofPlan)

### 4.7.3 NOVEL_CONNECTED
NOVEL_CONNECTED means “not near-duplicate, but connected to existing knowledge”.

Route to Step 5 and provide:
- `KGR`
- `best_match context`
- neighbor lists for grounding

### 4.7.4 NOVEL_ORPHAN
NOVEL_ORPHAN means connectedness below `τ_orphan` to BOTH bases under declared snapshot bindings.

Route to **Step 4.7.5 (Orphan Handling)**.

---

## 4.7.5 Orphan handling (Evidence-only)

This section is invoked only when `class = NOVEL_ORPHAN`.

### 4.7.5.1 Required inputs
- `co_id`
- `risk_class ∈ {LOW, MED, HIGH}`
- `suspected_domain.tags[]` (may be empty)
- `routing.reason` (string)

### 4.7.5.2 OrphanIncidentRecord (OIR)
Step 4 SHOULD emit an `OIR` with:
- `co_id`
- `risk_class`
- `suspected_domain.tags[]`
- `routing.reason`
- `kgr_ref` (pointer to KGR)
- `policy.orphan_consensus.protocol_ref`

### 4.7.5.3 Routing rules by risk class
- `LOW`: auto-queue for later clustering; no human quorum required.
- `MED`: require at least `policy.orphan_consensus.quorum.min_agents` agent review.
- `HIGH`: require human quorum `policy.orphan_consensus.quorum.min_humans` and agent quorum.

### 4.7.5.4 Human role / consensus
If `policy.orphan_consensus.enabled=true`, the downstream consensus MUST follow `policy.orphan_consensus.protocol_ref`.
Step 4 MUST NOT decide truth; it only routes and emits evidence artifacts.

## 4.8 Failure Modes (No-Guessing)

### 4.8.1 Missing policy binding
If `policy_config_ref` missing:
- emit `KGR.class = BLOCKED_POLICY_MISSING`
- do not route to Step 6
- route to Step 5 with an EvidenceGapPlan stub (policy missing)

### 4.8.2 Missing snapshot binding
If required snapshot binding fields are missing:
- emit `KGR.class = BLOCKED_INDEX_UNBOUND`
- do not perform heuristic retrieval
- route to Step 5 with EvidenceGapPlan stub (index unbound)

---

## 4.9 Guarantees
Step 4 v1.1.1 guarantees:
- deterministic classification under fixed policy config + snapshot bindings
- auditability: KGR ties result to exact base snapshots + implementations
- safe routing: no heuristic fallbacks when bindings are missing
- orphan handling is evidence-only and risk-aware

---

## 4.10 Multi-domain connectedness (Normative)
For claims that span multiple domains/elements, Step 4 MUST treat connectedness as a single scalar gate:
- compute `C_connect` as defined (4.2.3) against candidates
- select `best_match` by tie-breaker (4.5.1)

No special multi-domain

---

## 4.11 Side outputs (Appendix D) (Required)

Step 4 MUST emit side outputs per **Appendix D — Side Outputs & Banks Contract v0.1**:
- `Appendix_D_SideOutputs_Banks_Contract_v0.1.md`
- SideRecord template example: `Appendix_D_SideRecord_Template_v0.1.yaml`

**Constraint:** this section adds an additional output channel (`side_records[]`) and mapping. It does **not** modify Step 4’s forward routing logic.

### 4.11.1 When to emit SideRecord
For each processed claim, Step 4 MUST emit a SideRecord if:
- routing ends in `DROP_DEFER` (fails novelty/connectedness gate), OR
- routing ends in `REWRITE_REQUIRED`, OR
- routing ends in any `BLOCKED_*` condition.

If routing ends in `FORWARD_TO_STEP5`, emitting a SideRecord is OPTIONAL (traceability).

### 4.11.2 SideRecord mapping (normative)
SideRecord MUST be produced with:

- `step_id = STEP4`
- `input_ref` MUST link to:
  - `claim_id`, and
  - the `index_snapshot_binding` produced in Step 3 (or equivalent pointer)
- `normalized_claim` MUST equal Step 4’s normalized claim text.

**Status mapping**
- If `DROP_DEFER` or `REWRITE_REQUIRED` → `status = REJECTED_NOVELTY`
- If any `BLOCKED_*` → `status = INSUFFICIENT` and `reason_codes` MUST include the blocking code.

**scores{} mapping (minimum)**
`scores` MUST include:
- `novelty_score`
- `connectedness_score`
- near-duplicate indicators (at least one of):
  - `simhash_distance` (or similarity)
  - `minhash_jaccard_est` (or LSH hit indicator)
- KGR summary fields:
  - `kgr_node_count`
  - `kgr_edge_count`
  - `kgr_seed_refs_count`

If Step 4 assigns a novelty cluster:
- `cluster_ref.cluster_id`, `cluster_ref.cluster_size`, `cluster_ref.w_indep` SHOULD be included (if known at this stage; otherwise defer to Appendix C adapter binding).

**reason_codes[] mapping (minimum)**
For `REJECTED_NOVELTY`, `reason_codes` MUST include one or more of:
- `novelty_below_threshold`
- `connectedness_below_threshold`
- `near_duplicate_cluster`
- `rewrite_required` (only if routing is REWRITE_REQUIRED)

For `BLOCKED_*`, `reason_codes` MUST include:
- `blocked_context_missing` OR `blocked_policy_missing` OR `blocked_index_snapshot_missing`.

**distance_to_pass mapping**
- If numeric gate fails: provide distance for the **nearest failing constraint**:
  - `distance_to_pass = {kind:numeric, metric:<novelty|connectedness>, threshold:<tau>, value:<score>, margin:(value - tau)}`
  - margin MUST be negative when failing.
- If categorical (`REWRITE_REQUIRED` or `BLOCKED_*`):
  - `distance_to_pass = {kind:categorical, miss:<routing_label>}`

**recommended_protocol mapping (default)**
- If `near_duplicate_cluster` dominates → `recommended_protocol = PARTNER_DEV` (differentiate hypothesis / add new evidence hooks).
- If `connectedness_below_threshold` dominates → `recommended_protocol = PARTNER_DEV` (build bridges/justification).
- If `novelty_below_threshold` dominates → `recommended_protocol = CREATIVE` (reframe / change angle).
- If `REWRITE_REQUIRED` dominates → `recommended_protocol = LEARNING`.
- If `BLOCKED_*` → `recommended_protocol = PARTNER_DEV` (binding repair) or `LEARNING` (missing fields).

### 4.11.3 Reopen triggers (minimum)
Step 4 SideRecords SHOULD include at least:
- `HUMAN_CHALLENGE`
- `NEW_SOURCE_CLUSTER` (if cluster grows or independence improves)
- `POLICY_CHANGED` (novelty thresholds change)

---

