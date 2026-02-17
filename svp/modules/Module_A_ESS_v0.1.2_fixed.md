# Module A — Evidence Strength Scoring (ESS) v0.1.2
**Status:** Normative (startup defaults; calibration expected)  
**Role:** converts evidence artifacts into a deterministic `strength ∈ [0..100]` and `confidence ∈ {weak, med, high}`, then supports aggregation into `SystemDelta` in **Step 6**.

---

## A.0 Purpose
ESS provides a reproducible scoring function:

- Input: a single evidence item `E_*` (e.g., `E_text`, `E_sim`, `E_obs`, …) with declared metadata.
- Output: `strength` and `confidence`, plus `excluded_reason` if the evidence is not admissible.

ESS MUST be used by Step 6 for weighting evidence contributions.

---

## A.0.1 Policy Binding (Required)
ESS MUST be policy-bound:

- `policy_config_ref` (REQUIRED)  
- `policy_config_hash` (RECOMMENDED)  
- `ess_version = "0.1.2"` (REQUIRED; must match)  

Policy MUST define (at minimum):
- `base_strength_table[evidence_class] → base ∈ [0..100]`
- `confidence_thresholds` (strength→{weak,med,high})
- `time_ref_days` for `horizon_class ∈ {short, medium, long}`
- `time_default_factor_when_missing ∈ [0..1]`
- `rtt.eps` (REQUIRED iff `E_sim` enabled)
- `rtt.cycles_ref[horizon_class][risk_class]` (REQUIRED iff `E_sim` enabled)

Policy SHOULD define (startup defaults permitted):
- `independence.triplet_weights` (defaults: `{data:0.50, method:0.25, code:0.25}`)
- `provenance.cluster_alpha` (default `1.0`)
- `provenance.unknown_cluster_penalty ∈ [0..1]` (default `0.50`)
- `independence.unknown_independence_penalty ∈ [0..1]` (default `0.50`)

**No-Guessing:** if `policy_config_ref` or required tables are missing → `BLOCKED_POLICY_MISSING`.  
ESS MUST NOT silently fallback to embedded defaults.

---

## A.1 Inputs (Normative)

Each evidence item passed to ESS MUST expose:

### A.1.1 Common fields
- `evidence_id`
- `evidence_class ∈ {E_text, E_ref, E_case, E_obs, E_sim, E_bio}` (extensible by policy)
- `scope_fit.in_scope ∈ {true,false}`
- `quality ∈ [0..1]` (or deterministic mapping from a quality tier)

Independence inputs (one of the following paths MUST be used deterministically):
- Path A (scalar): `independence ∈ [0..1]`
- Path B (triplet): `independence_triplet.{data, method, code} ∈ [0..1]`

Provenance / clustering inputs (OPTIONAL but supported; used for cluster penalty):
- `provenance.source_id` (stable string id; OPTIONAL)
- `provenance.cluster_id` (OPTIONAL)
- `provenance.cluster_size` (OPTIONAL; integer ≥ 1)
- `provenance.w_indep` (OPTIONAL; if provided, takes precedence over cluster_size-derived penalty)

Time / validity inputs:
- `time_window_days` (optional)
- `horizon_class ∈ {short, medium, long}` (required for time normalization)
- `validity_passed ∈ {true,false}` (required; result of Step 6 Stage 3 class-profile checks)

### A.1.2 Class-specific fields
- For `E_sim` (if used):
  - `builder_vs_critic_delta` (aka `ab_delta`)
  - `stress_suite_pass_rate`
  - `cycles_run`

---

## A.1.3 Exclusion rules (Normative)

1) If `scope_fit.in_scope = false` → evidence MUST NOT contribute:
- `strength = 0`
- `excluded_reason = "OUT_OF_SCOPE"`

2) If `validity_passed = false` → evidence MUST NOT contribute:
- `strength = 0`
- `excluded_reason = "CLASS_PROFILE_FAILED"`

---

## A.2 Base Strength by Evidence Class (Policy-bound)

ESS defines a base strength table:

- `base_strength_table[evidence_class] → base ∈ [0..100]`

**Startup defaults** (policy SHOULD load these as initial values):
- `E_text = 15`
- `E_ref  = 25`
- `E_case = 35`
- `E_sim  = 45`
- `E_obs  = 55`
- `E_bio  = 60` (if enabled)

**Normative constraint:** base strengths MUST be loaded from policy (not hardcoded).  
If missing → `BLOCKED_POLICY_MISSING`.

---

## A.3 Quality Factor (Q)

`Q ∈ [0..1]` MUST be provided or deterministically derived.

Default:
- `Q = quality`

If policy uses tiers, the tier→Q mapping MUST be versioned and logged.

---

## A.4 Independence Factor (I)

`I ∈ [0..1]` MUST be derived deterministically.

ESS decomposes independence into two factors:
- `I_triplet` — methodological/data/code independence (within one source/cluster)
- `I_cluster` — cross-source independence penalty based on provenance clustering

### A.4.1 I_triplet (within-source independence)
Let policy define weights `w_data, w_method, w_code` (startup defaults `{0.50, 0.25, 0.25}`).

Deterministic derivation:
- If `independence_triplet` is present:
  - `I_triplet = clip01(w_data*data + w_method*method + w_code*code)`
- Else if scalar `independence` is present:
  - `I_triplet = clip01(independence)`
- Else:
  - `I_triplet = policy.independence.unknown_independence_penalty`
  - ESS MUST emit reason code `INDEPENDENCE_MISSING_DEFAULTED` (see A.11).

### A.4.2 I_cluster (cluster penalty)
Cluster penalty is policy-defined to prevent pseudo-amplification from many dependent sources.

Deterministic derivation (precedence order):
1) If `provenance.w_indep` is present:
   - `I_cluster = clip01(provenance.w_indep)`
2) Else if `provenance.cluster_size` is present:
   - let `alpha = policy.provenance.cluster_alpha` (default `1.0`)
   - `I_cluster = clip01( 1 / (cluster_size ^ alpha) )`
3) Else:
   - `I_cluster = policy.provenance.unknown_cluster_penalty`
   - ESS MUST emit reason code `CLUSTER_INFO_MISSING_DEFAULTED` (see A.11).

### A.4.3 Final I
- `I = clip01(I_triplet * I_cluster)`

Definitions:
- `clip01(x) = min(1, max(0, x))`

---

## A.5 Time / Horizon Normalization (T)

Time factor `T ∈ [0..1]` depends on `horizon_class`.

Let `T_ref` be policy-defined via `time_ref_days[horizon_class]` (startup defaults permitted):
- `short: 7`
- `medium: 30`
- `long: 180`

If `time_window_days` exists:
- `T = clip01(time_window_days / T_ref)`

If `time_window_days` missing:
- `T = policy.time_default_factor_when_missing`
- ESS SHOULD emit reason code `TIME_WINDOW_MISSING_DEFAULTED` (see A.11).

Definitions:
- `clip01(x) = min(1, max(0, x))`

---

## A.6 Special Terms for E_sim (Stability & Stress)

For `E_sim`, ESS introduces additional multipliers:

### A.6.1 Stability
- `S_stab = clip01(1 - builder_vs_critic_delta / eps)`

Where `eps = policy.rtt.eps` (policy-bound; see Module B / RTT). If missing and `evidence_class=E_sim` → `BLOCKED_POLICY_MISSING`.

### A.6.2 Stress
- `S_stress = stress_suite_pass_rate` (already in `[0..1]`)

### A.6.3 Cycles
- `S_cycles = clip01( log(1 + cycles_run) / log(1 + cycles_ref) )`

Where `cycles_ref = policy.rtt.cycles_ref[horizon_class][risk_class]` (policy-bound; see Module B / RTT). If missing and `evidence_class=E_sim` → `BLOCKED_POLICY_MISSING`.

---

## A.7 Final Strength Formula (Deterministic)

Define:
- `base = base_strength_table[evidence_class]`

### A.7.1 Non-simulation evidence
For all `E_*` except `E_sim`:
- `strength = round_half_up( base * Q * I * T )`

### A.7.2 Simulation evidence
For `E_sim`:
- `strength = round_half_up( base * Q * I * T * S_stab * S_stress * S_cycles )`

**Determinism requirement:**
- rounding MUST be `round_half_up` (policy MAY override but must declare `rounding_mode`).

Definitions:
- `round_half_up(x) = floor(x + 0.5)` for `x ≥ 0` (all ESS strengths are nonnegative).

---

## A.8 strength → confidence mapping (Policy-bound)

Policy MUST define confidence thresholds. Startup defaults:
- `strength < 50` → `weak`
- `50 ≤ strength < 75` → `med`
- `strength ≥ 75` → `high`

---

## A.9 Aggregation Interface into SystemDelta (Step 6)

ESS provides the weighting function `W(e)` used by Step 6:

- `W(e) = sign(e) * strength(e)`

Where `sign(e) ∈ {-1, 0, +1}` is produced by Step 6 logic (class/claim semantics), not by ESS.

### A.9.1 Required aggregation inputs
Step 6 aggregation MUST supply:
- `horizon_class ∈ {short, medium, long}`
- `risk_class ∈ {LOW, MED, HIGH}`
- `element_profile_passed[element] ∈ {true,false}` for each element

### A.9.2 No-Guessing override (per element)
If `element_profile_passed[element] = false`:
- that element’s aggregated sign MUST be `0` regardless of evidence weights.

### A.9.3 Threshold usage
The final decision threshold `Θ` is defined in Module B (RTT) by horizon/risk.

---

## A.10 ESS Output Contract

ESS MUST output:

- `strength ∈ [0..100]`
- `confidence ∈ {weak, med, high}`
- `excluded_reason?`
- `ess_version`, `policy_config_ref`, `policy_config_hash?`
- intermediate multipliers used:
  - `Q`
  - `I_triplet`, `I_cluster`, `I`
  - `T`
  - for `E_sim` also `S_stab`, `S_stress`, `S_cycles`
- `reasons[]` (OPTIONAL; deterministic informational codes; see A.11)

---

## A.11 Deterministic reason codes (Normative)
ESS MAY emit `reasons[]` to indicate conservative defaults or exclusions. Codes are deterministic strings:

Exclusions:
- `OUT_OF_SCOPE` (when `scope_fit.in_scope=false`)
- `CLASS_PROFILE_FAILED` (when `validity_passed=false`)

Conservative defaults (policy-driven):
- `INDEPENDENCE_MISSING_DEFAULTED`
- `CLUSTER_INFO_MISSING_DEFAULTED`
- `TIME_WINDOW_MISSING_DEFAULTED`

**No-Guessing:** ESS MUST NOT fabricate provenance or independence fields; it may only apply policy-defined conservative defaults.

---
