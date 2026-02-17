# Appendix D — Side Outputs & Banks Contract v0.1 (SFI / SVP‑EVAL)

**Status:** Normative appendix (contract)  
**Revision:** v0.1 (2026-02-14)  
**Applies to:** STEP 2–STEP 6 (optionally STEP 1)  
**Goal:** define **non-forward** outputs at each step (“side outputs”), and rules for storing/reusing rejected/insufficient material in a working environment (SFI).

---

## D.1 Scope

SFI runs a 6-step filtering pipeline. The *forward* path produces candidate TRUE ideas at STEP 6.  
This appendix defines **mandatory side outputs** produced by each step (starting at STEP 2 by default) and their storage/usage contracts:

- Side outputs MUST be returned to the **personal AI agent** with scores and reason codes.
- Side outputs MUST be available in the **working environment** for re-check, reuse, and cost-saving lookup (error/hypothesis banks).
- A human MAY challenge any record (TRUE or not) and trigger re-evaluation.

> Constraint: this appendix **does not change** the structure of STEP 1–6. It only specifies additional outputs and links.

---

## D.2 Normative language

**MUST / SHOULD / MAY** are normative.

---

## D.3 Core rule: every step returns two streams

For each STEP _i_:

1) **Forward output** (existing): payload that passes to STEP _i+1_.
2) **Side output** (new): list of SideRecord entries.

**Contract:**
- STEP _i_ MUST emit `side_records[]` for all processed claims/clusters/hypotheses that fail gating **or** require deferral due to gaps.
- A record that passes forward MAY also emit a SideRecord (e.g., for traceability), but this is OPTIONAL unless required by the step spec.

Minimal interface (conceptual):
`STEP_i(input) -> { forward_payload? , side_records[] }`

---

## D.4 SideRecord v0.1 (normative fields)

### D.4.1 Required identifiers
- `side_record_id` (unique within `run_id`)
- `run_id` (pipeline run)
- `step_id` ∈ {`STEP2`,`STEP3`,`STEP4`,`STEP5`,`STEP6`} (STEP1 OPTIONAL)
- `timestamp_utc`

### D.4.2 Required linkage
- `input_ref` (pointer to original fragment/source/claim record)
- `normalized_claim` (canonicalized claim text)

### D.4.3 Decision/status
- `status` ∈
  - `REJECTED_CHAOS` (STEP 2–3)
  - `REJECTED_NOVELTY` (STEP 4)
  - `INSUFFICIENT` (STEP 5–6; evidence/provenance/coverage gaps)
  - `UNKNOWN_SYN` (STEP 6; not enough support to assert TRUE/FALSE)
  - `HYPOTHESIS_POS` / `HYPOTHESIS_NEG` (see D.6; optional classification)

> Legacy status `inconclusive` MUST NOT be used.

### D.4.4 Scores & reasons
- `scores{}`: step-specific numeric scores (chaos, novelty, connectedness, evidence_strength, etc.)
- `reason_codes[]`: step-specific reasons; for evidence/provenance gaps MUST use Appendix B codes (e.g., `provenance_missing_fields`).

### D.4.5 Distance to pass (critical)
- `distance_to_pass`: numeric or structured distance from the nearest pass threshold.
  - If a threshold is categorical, distance MUST be represented as `{kind: categorical, miss: <label>}`.
  - If numeric, MUST be a signed float: negative = failed by margin; positive = passed by margin.

### D.4.6 Agent protocol suggestion
- `recommended_protocol` ∈
  - `CREATIVE` (reframe / ideation)
  - `LEARNING` (simplify / extract / paraphrase)
  - `PARTNER_DEV` (novelty/hypothesis co-development)
  - `EVIDENCE` (collect references / measurements)
  - `SIMULATION` (Protocol S)
  - `FORMALIZE` (Protocol U)

Personal AI agent MAY override `recommended_protocol`, but MUST log an override event with `override_reason`.

### D.4.7 Reopen triggers
- `reopen_triggers[]` MUST be present and MAY be empty.
- Each trigger MUST declare the condition that should cause re-evaluation (D.7).

---

## D.5 Step-specific side output rules

### D.5.1 STEP 2–3 (Chaos gates)
If chaos gating fails:
- `status = REJECTED_CHAOS`
- `scores` MUST include the computed chaos score(s).
- `recommended_protocol` SHOULD be `LEARNING` (or `CREATIVE` if the claim is intelligible but noisy).
- `distance_to_pass` MUST refer to the chaos threshold.

### D.5.2 STEP 4 (Novelty & Connectedness gate)
If novelty/connectedness gating fails:
- `status = REJECTED_NOVELTY`
- `scores` SHOULD include: novelty score, connectedness score, near-dup indicators.
- If near-dup clustering exists, SideRecord SHOULD include:
  - `cluster_ref` (cluster_id/cluster_size)
- `recommended_protocol` SHOULD be `PARTNER_DEV` (or `CREATIVE` if the hypothesis needs reframing).
- `distance_to_pass` MUST reference the relevant thresholds.

### D.5.3 STEP 5–6 (Evidence builder / Evaluation)
If evidence/provenance/coverage is insufficient:
- `status = INSUFFICIENT`
- `reason_codes` MUST contain Appendix B codes.
- `recommended_protocol` SHOULD be `EVIDENCE` or `SIMULATION` depending on which gap dominates.
- `distance_to_pass` MUST reflect missing evidence mass, coverage deficit, or penalty margin.

If STEP 6 cannot assert TRUE/FALSE but is not blocked by formal gaps:
- `status = UNKNOWN_SYN`
- `recommended_protocol` SHOULD be `PARTNER_DEV` or `EVIDENCE`.

---

## D.6 Hypothesis/“error” semantics (banks)

### D.6.1 Hypothesis Bank (HB)
HB stores all SideRecords (and optionally forward records) as **re-evaluable hypotheses**.

HB entries MUST include:
- the latest `status`
- `H_rank ∈ [-1, +1]` (priority axis)
- `validity_scope` (domain/context tags)
- `supporting_refs[]` / `counter_refs[]` (when available)

### D.6.2 Error Bank (EB) as a view
EB is a **view** over HB where:
- `H_rank` is low (e.g., `< -0.6`) AND
- there exists at least one falsifying artifact (`counter_refs[]`).

**Important:** EB entries MUST remain re-openable; an “error” is a negative hypothesis under current data.

### D.6.3 Gap Bank (GB)
GB stores recurring insufficient patterns:
- missing provenance fields
- weak independence
- missing coverage slices
- simulation-required triggers

GB entries SHOULD store reusable “fix plans” and links to Appendix B codes.

### D.6.4 Energy-saving lookup rule (normative)
Before running expensive evaluation or simulation, the system SHOULD query:
1) EB/HB for known counterexamples or near-identical rejected patterns,
2) GB for standard remediation plans.

If lookup yields a high-confidence match, the system MAY short-circuit expensive compute and emit a SideRecord referencing the match.

---

## D.7 Reopen triggers (minimum set)

A SideRecord MAY be re-opened when any of the following occurs:

- `NEW_SOURCE_CLUSTER`: new independent sources increase `cluster_size` or `w_indep`.
- `POLICY_CHANGED`: policy_id / key thresholds changed.
- `COUPLING_GRAPH_CHANGED`: coupling_graph_version/hash changed.
- `TTL_EXPIRED`: time-based refresh in fast-evolving domains.
- `HUMAN_CHALLENGE`: human issues a ChallengeEvent (D.8).

Each trigger MUST include a condition and a recommended protocol.

---

## D.8 Human challenge loop (ChallengeEvent)

A human MAY challenge any record (TRUE, rejected, insufficient, unknown).

ChallengeEvent MUST include:
- `target_record_ref`
- `objection_text`
- optional `new_sources[]`
- `demand` ∈ {`RETEST`,`REFRAME`,`FORMALIZE`}

On ChallengeEvent:
- personal AI agent MUST respond as “interested” (non-dismissive)
- the system MUST generate a re-evaluation plan (protocol selection + expected artifacts)
- re-evaluation MUST result in a new SideRecord (or forward record) linked to the challenge.

---

## D.9 Interop notes (existing appendices/protocols)

- Appendix B: evidence gap codes (mandatory for `INSUFFICIENT`)
- Appendix C: adapter provenance rules (source_id/cluster_id/cluster_size/w_indep)
- Protocol S: simulation evidence
- Protocol U: formalization/orphan consensus
