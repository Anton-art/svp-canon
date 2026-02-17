# Module B — Robustness Threshold Table (RTT) v0.1.2
**Status:** Normative (calibration expected; no embedded defaults)  
**Role:** provides policy-bound thresholds `Θ` for SystemDelta aggregation and defines the robustness contract for `E_sim`
used by Step 6 (harm/benefit robustness, time-lag/hidden debt).

---

## B.0 Purpose
RTT defines:

1) `Θ` thresholds for converting aggregated evidence weights `V_e` into element signs `{-1,0,+1}`.  
2) minimal requirements for `E_sim` when simulation is required (especially for flow-affecting claims).  
3) `robustness.verdict` mapping compatible with Step 6:
- `STABLE`, `UNSTABLE`, `INSUFFICIENT`

---

## B.0.1 Policy Binding (Required)
RTT MUST be policy-bound:

- `policy_config_ref` (REQUIRED)
- `policy_config_hash` (RECOMMENDED)
- `rtt_version = "0.1.2"` (REQUIRED; must match)

Policy MUST define (at minimum):
- `theta_table[horizon_class][risk_class]`
- `eps`
- `min_stress_pass[horizon_class][risk_class]`
- `min_cycles[horizon_class][risk_class]`
- `cycles_ref[horizon_class][risk_class]`

Optional (policy-enabled) resampling robustness parameters:
- `resampling.enabled ∈ {true,false}` (default false)
- `resampling.bootstrap_runs_min`
- `resampling.ci_level`
- `resampling.sign_stability_tau`
- `resampling.influence_tau`
- `resampling.influence_topk`

**No-Guessing:** if any REQUIRED thresholds are missing → `verdict=INSUFFICIENT` and include `BLOCKED_POLICY_MISSING`.

---
## B.1 Θ Threshold Table for Aggregation (Policy-bound)

RTT provides `Θ = theta_table[horizon_class][risk_class]`.

**Interpretation:**
- element sign = `+1` iff `V_e ≥ +Θ`
- element sign = `-1` iff `V_e ≤ -Θ`
- element sign = `0` otherwise

**Normative constraint:** theta values MUST come from policy (not embedded here).  
RTT MUST log the selected `theta` in `RTT_Output`.

---

## B.2 Minimum Requirements for E_sim (Policy-bound)

When simulation is required, RTT defines minimum simulation thresholds.

Policy MUST define:
- `eps` (non-negative)
- `min_stress_pass[horizon_class][risk_class]` (0..1)
- `min_cycles[horizon_class][risk_class]` (integer ≥ 0)
- `cycles_ref[horizon_class][risk_class]` (integer ≥ 0; used by ESS in `S_cycles`)

RTT MUST treat all thresholds as policy-derived (no embedded defaults).

Policy MAY define stricter requirements via a higher-stakes profile (see B.12).

---
## B.3 Robustness Verdict Enum (Normative)

RTT defines:
- `robustness.verdict ∈ {STABLE, UNSTABLE, INSUFFICIENT}`

RTT MUST select thresholds from policy (and log them in `thresholds_used`):
- `eps`
- `min_stress_pass`
- `min_cycles`
- `cycles_ref` (logged; not used in the baseline mapping below)

### B.3.1 E_sim field interpretation
- `ab_delta` MUST be the absolute builder-vs-critic disagreement:
  - `ab_delta = abs(builder_vs_critic_delta)`
- `stress_pass_rate ∈ [0,1]`
- `cycles_run` is an integer count of simulation cycles executed
- `time_lag_ok` is computed deterministically (B.6 and B.10)

### B.3.2 Mapping rules (deterministic)
- `INSUFFICIENT` if:
  - `simulation_required=true` but `E_sim` is missing → add reason `MISSING_E_SIM`, OR
  - any required `E_sim` field is missing (see B.11) → add `MISSING_E_SIM_FIELD:<field_path>`

- `UNSTABLE` if any holds (add the corresponding reasons):
  - `ab_delta > eps` → `AB_DELTA_GT_EPS`
  - `stress_pass_rate < min_stress_pass` → `STRESS_BELOW_MIN`
  - `cycles_run < min_cycles` → `CYCLES_BELOW_MIN`
  - `time_lag_ok = false` → `TIME_LAG_POLICY_FAILED` unless the cause is debt-channels (see B.6)

- `STABLE` otherwise

RTT MUST output `reasons[]` codes explaining the verdict deterministically.

---
## B.4 Robust Harm Conditions (FALSE_SYN)

For `FALSE_SYN` to be considered ROBUST harm:

- If claim is flow-affecting: `simulation_required` MUST be true and `E_sim` is REQUIRED and MUST satisfy:
  - `robustness.verdict == STABLE`
- If claim is not flow-affecting: robustness MAY be satisfied by non-sim evidence if policy allows, but `simulation_required` becomes mandatory when `risk_class=HIGH` under stricter profiles.

---

## B.5 Robust Benefit Conditions (Systemic TRUE_SYN on flow-affecting)

For systemic `TRUE_SYN` to be considered ROBUST benefit when flow-affecting:

- `simulation_required = true`
- `E_sim` is REQUIRED
- `robustness.verdict == STABLE`
- `borrowed_benefit = false`
- `time_lag_ok = true` within the horizon

If borrowed benefit is detected, the system MUST NOT output systemic TRUE_SYN unless scope is narrowed to exclude debt channels.

---

## B.6 Time-lag / Hidden Debt Rule (Deterministic, baseline)

RTT defines a baseline deterministic “hidden debt” trigger.

Let `debt_channels[] = E_sim.borrowed_benefit.debt_channels[]` (may be empty).

If `debt_channels[]` is non-empty:
- `borrowed_benefit = true`
- `time_lag_ok = false`
- RTT MUST append reason `BORROWED_BENEFIT_DEBT_CHANNELS`
- systemic `TRUE_SYN` is forbidden unless scope narrowing excludes those channels.

If `debt_channels[]` is empty:
- `borrowed_benefit = false`
- `time_lag_ok = true` (baseline; may be overridden by policy, see B.10)

---
## B.7 RTT Output Contract (Completion)

RTT MUST provide:
- `theta_table` lookup access
- `eps`, `min_stress_pass`, `min_cycles`, `cycles_ref`
- the robustness verdict mapping rules (B.3)
- (if policy-enabled) resampling robustness rules (B.7.1)

All values MUST be traceable to:
- `policy_config_ref`
- `policy_config_hash` (if provided)
- `rtt_version`
- `thresholds_used` (eps/min_stress_pass/min_cycles/cycles_ref and selected Θ)
- (if policy-enabled) `resampling_thresholds_used`

RTT MUST emit a machine-readable output object `RTT_Output` (see B.9).

---
## B.7.1 Resampling Robustness (Bootstrap/Jackknife) (Policy-enabled, Normative)

### B.7.1.1 Purpose
This section adds an optional, deterministic robustness layer driven by policy.  
It detects "single-point-of-failure" conclusions that rely on one evidence cluster.

### B.7.1.2 Inputs (provided via RTT_Input.resampling)
When `resampling.enabled=true`, caller MAY provide:
- `Ve_samples[]`: bootstrap samples of aggregated evidence weight `V_e` (same target as the caller is evaluating)
- `influence[]`: jackknife influence list, each item `({cluster_id, influence})`
- `bootstrap_runs_actual`: number of bootstrap runs that produced `Ve_samples[]`

**No-Guessing:** RTT MUST NOT fabricate samples.  
If `resampling.enabled=true` but `Ve_samples[]` is missing → resampling verdict = `INSUFFICIENT` and add reason `RESAMPLING_INPUT_MISSING`.

### B.7.1.3 Deterministic computation
Given selected `theta`:

1) `N = len(Ve_samples)`
2) If `N < resampling.bootstrap_runs_min` → resampling verdict = `INSUFFICIENT` and add reason `BOOTSTRAP_INSUFFICIENT_RUNS`.

3) Per-sample sign:
- `sign(v) = +1` iff `v ≥ +theta`
- `sign(v) = -1` iff `v ≤ -theta`
- `sign(v) =  0` otherwise

4) `sign_mode = mode(sign(Ve_samples[i]))` with deterministic tie-break: `+1 > 0 > -1`  
5) `sign_stability = count(sign==sign_mode) / N`

6) CI over `V_e` using deterministic quantiles (nearest-rank method):
- sort samples ascending: `S = sort(Ve_samples)`
- define `q(p)` for `p∈(0,1]` as `S[ceil(p*N)]` (1-indexed; clamp to `[1..N]`)
- let `p_low = (1 - resampling.ci_level)/2`, `p_high = 1 - p_low`
- `ci_low = q(p_low)`, `ci_high = q(p_high)`

7) CI-consistency test:
- if `ci_low ≥ +theta` → CI implies `+1`
- else if `ci_high ≤ -theta` → CI implies `-1`
- else if `(-theta < ci_low) AND (ci_high < +theta)` → CI implies `0`
- else → CI implies `MIXED`

8) Influence:
- `influence_max = max(influence[i].influence)` (or 0 if none provided)
- `influence_topk`: top `K` by `influence` desc; ties broken by `cluster_id` asc; `K = resampling.influence_topk`

### B.7.1.4 Resampling verdict mapping (deterministic)
- `INSUFFICIENT` if:
  - `Ve_samples[]` missing, OR
  - `N < resampling.bootstrap_runs_min`

- `UNSTABLE` if any holds (add the corresponding reasons):
  - `sign_stability < resampling.sign_stability_tau` → `SIGN_STABILITY_BELOW_TAU`
  - `CI implies MIXED` → `CI_OVERLAPS_THRESHOLD`
  - `influence_max > resampling.influence_tau` → `INFLUENCE_ABOVE_TAU`

- `STABLE` otherwise

### B.7.1.5 Merge with baseline (E_sim) robustness
If baseline verdict exists (B.3) and resampling verdict exists, RTT MUST merge:
- order: `STABLE > UNSTABLE > INSUFFICIENT`
- merged verdict = minimum under this order (i.e., any `INSUFFICIENT` dominates; else any `UNSTABLE` dominates)

RTT MUST append resampling reasons to `reasons[]`.

---
## B.8 RTT Input Contract (Normative)

RTT MUST accept a single evaluation request `RTT_Input` containing:

- `policy_config_ref` (REQUIRED)
- `policy_config_hash` (RECOMMENDED)
- `rtt_version` (REQUIRED, must equal `"0.1.2"`)
- `horizon_class ∈ {short, medium, long}` (REQUIRED)
- `risk_class ∈ {LOW, MED, HIGH}` (REQUIRED)
- `claim_flow_affecting ∈ {true,false}` (REQUIRED)
- `simulation_required ∈ {true,false} | null` (REQUIRED; may be computed by RTT if caller sets it null)
- `E_sim` object (REQUIRED iff `simulation_required=true`)

Optional higher-stakes profile selector:
- `stakes_profile_ref` (OPTIONAL; if present, MUST match a profile defined by policy; see B.12)

Optional (policy-enabled) resampling robustness input:
- `resampling` (OPTIONAL):
  - `enabled ∈ {true,false}` (if omitted, RTT uses policy default)
  - `Ve_samples[]` (OPTIONAL; REQUIRED to run resampling when enabled=true)
  - `bootstrap_runs_actual` (OPTIONAL)
  - `influence[]` (OPTIONAL; jackknife cluster influences)

### B.8.1 simulation_required triggers (if caller leaves it null)
If caller does not supply `simulation_required`, RTT MUST compute it deterministically:

- `simulation_required = true` if (`claim_flow_affecting=true` AND `risk_class=HIGH`)
- `simulation_required = true` if (`claim_flow_affecting=true` AND `horizon_class ∈ {medium,long}` AND `risk_class ∈ {MED,HIGH}`)
- `simulation_required` MAY be forced by a higher-stakes profile reference (see B.12)
- otherwise `simulation_required = false`

**No-Guessing:** if `simulation_required=true` and `E_sim` is missing → `verdict=INSUFFICIENT` and include `MISSING_E_SIM`.

---
## B.9 RTT Output Object (Normative)

RTT MUST output `RTT_Output`:

- `policy_config_ref`
- `policy_config_hash?`
- `rtt_version`
- `horizon_class`, `risk_class`, `claim_flow_affecting`
- `simulation_required`
- `stakes_profile_ref?` (if provided in input)
- `theta` (selected Θ)
- `thresholds_used`:
  - `eps`
  - `min_stress_pass`
  - `min_cycles`
  - `cycles_ref`

- `resampling_thresholds_used` (OPTIONAL; only when resampling is enabled by policy or input):
  - `bootstrap_runs_min`
  - `ci_level`
  - `sign_stability_tau`
  - `influence_tau`
  - `influence_topk`

- `observed` (if `E_sim` present):
  - `ab_delta`
  - `stress_pass_rate`
  - `cycles_run`
  - `borrowed_benefit`
  - `debt_channels[]` (derived from `E_sim.borrowed_benefit.debt_channels[]`)
  - `time_lag_ok` (boolean)

- `resampling_observed` (OPTIONAL; only if resampling enabled and `Ve_samples[]` present):
  - `enabled`
  - `bootstrap_runs_actual`
  - `N_samples`
  - `ci_level`
  - `ci_low`, `ci_high`
  - `sign_mode ∈ {-1,0,+1}`
  - `sign_stability`
  - `influence_max`
  - `influence_topk[]` (list of `{cluster_id, influence}`)

- `verdict ∈ {STABLE, UNSTABLE, INSUFFICIENT}`
- `reasons[]` (deterministic string codes)

---
## B.10 Time-lag / borrowed benefit formalization (Extended, policy-hook)

RTT MUST compute:
- `debt_channels[] = E_sim.borrowed_benefit.debt_channels[]` (may be empty)
- `borrowed_benefit = true` iff `debt_channels[]` is non-empty
- baseline `time_lag_ok = true` iff `borrowed_benefit=false` (see B.6)

Policy MAY extend time-lag checks via:
- `time_lag_cases_required` (e.g., require `"SS_TIME_LAG"` in the stress suite)
- `time_lag_pass_min` (minimum pass rate for time-lag subset)

If policy declares time-lag requirements and they are not met:
- `time_lag_ok = false`
- verdict MUST be `UNSTABLE`
- reasons MUST include `TIME_LAG_POLICY_FAILED`

---
## B.11 Required fields for E_sim (Normative)

If `simulation_required=true`, `E_sim` MUST contain:
- `ab_delta`
- `stress_pass_rate`
- `cycles_run`
- `borrowed_benefit.debt_channels[]` (may be empty)

If any required field is missing:
- `verdict = INSUFFICIENT`
- `reasons[]` MUST include `MISSING_E_SIM_FIELD:<field_path>`

---
## B.12 Higher-stakes profile hook (Normative)

Policy MAY define stricter threshold profiles addressable by reference.

If `stakes_profile_ref` is provided in `RTT_Input`, RTT MUST:
- locate the referenced profile in policy
- apply the profile overrides for any of: `eps`, `min_stress_pass`, `min_cycles`, `cycles_ref`, and/or `theta`
- log `stakes_profile_ref` in `RTT_Output`

**No-Guessing:** if `stakes_profile_ref` is provided but not found in policy → `verdict=INSUFFICIENT` and include `BLOCKED_POLICY_MISSING`.

---
## B.13 Deterministic reason codes (Normative)

RTT MUST use the following reason codes (extendable by policy):

Policy / binding:
- `BLOCKED_POLICY_MISSING`

Baseline (E_sim):
- `MISSING_E_SIM`
- `MISSING_E_SIM_FIELD:<field_path>`
- `AB_DELTA_GT_EPS`
- `STRESS_BELOW_MIN`
- `CYCLES_BELOW_MIN`
- `BORROWED_BENEFIT_DEBT_CHANNELS`
- `TIME_LAG_POLICY_FAILED`

Resampling (policy-enabled):
- `RESAMPLING_INPUT_MISSING`
- `BOOTSTRAP_INSUFFICIENT_RUNS`
- `SIGN_STABILITY_BELOW_TAU`
- `CI_OVERLAPS_THRESHOLD`
- `INFLUENCE_ABOVE_TAU`

---