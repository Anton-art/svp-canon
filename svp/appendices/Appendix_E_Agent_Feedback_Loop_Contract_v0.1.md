# Appendix E — Agent Feedback Loop Contract v0.1 (SFI / SVP)

**Status:** Normative appendix (contract)  
**Revision:** v0.1 (2026-02-14)  
**Depends on:** Appendix D (Side Outputs & Banks), Appendix B (EvidenceGap codes), Protocol S, Protocol U  
**Goal:** define how **personal AI agents** receive, interpret, store, and act on *non-forward* outputs (SideRecords) produced by STEP 2–STEP 6.

---

## E.1 Scope

This appendix specifies the **Agent Feedback Loop** for working environments where:
- every STEP emits `side_records[]` per Appendix D,
- side_records are returned to a **personal AI agent** for user-specific actions (learning/creative/partner-dev),
- side_records and derived artifacts are stored in banks (HB/EB/GB) for energy-saving reuse.

**Constraint:** Appendix E does **not** change the STEP 1–6 structure, gates, or forward outputs.

---

## E.2 Normative language

**MUST / SHOULD / MAY** are normative.

---

## E.3 Interfaces

### E.3.1 Inputs to the agent
The agent consumes an **AgentFeedbackEnvelope** produced once per run or per step batch.

Minimum envelope:
- `run_id`
- `user_profile_ref` (stable pointer; not raw PII)
- `policy_ref` (PolicyPack id + hash)
- `side_records[]` (Appendix D SideRecords)

### E.3.2 Outputs from the agent
The agent emits:
- `AgentActionPlan` (recommended next actions + expected artifacts)
- optional `AgentOverrideEvent` entries (when overriding recommended_protocol)
- optional `ReopenRequest` (when triggering re-evaluation via reopen_triggers)

---

## E.4 Mandatory return-to-agent fields

For each SideRecord, the system MUST return at least:

- identifiers: `side_record_id`, `run_id`, `step_id`, `timestamp_utc`
- linkage: `input_ref`, `normalized_claim`
- decision: `status`
- scoring: `scores{}`, `distance_to_pass`
- explanation: `reason_codes[]`
- steering: `recommended_protocol`, `reopen_triggers[]`
- references: `links{}` (when present; MUST include Appendix B/C/D refs if applicable)

If any required field is missing:
- the system MUST mark the record as `INSUFFICIENT` (implementation gap) and include `reason_codes += [provenance_missing_fields]` **or** the closest available Appendix B code.
- PolicyPack MAY require fail-closed behavior (see PolicyPack `side_outputs.fail_closed_on_missing_side_records`).

---

## E.5 Protocol selection logic (agent-side)

The agent MUST map each SideRecord to one of the **agent protocols**:

- `LEARNING` — simplify, extract, paraphrase, request missing context fields
- `CREATIVE` — reframe, generate alternative angles, increase novelty
- `PARTNER_DEV` — co-develop hypothesis with user; refine connectedness; articulate testable form
- `EVIDENCE` — collect sources/measurements; fix provenance and coverage gaps
- `SIMULATION` — request or run Protocol S artifacts
- `FORMALIZE` — request or run Protocol U artifacts

### E.5.1 Default mapping by status
- `REJECTED_CHAOS` → `LEARNING` (default), MAY use `CREATIVE` if the claim is intelligible but noisy
- `REJECTED_NOVELTY` → `PARTNER_DEV` (default), MAY use `CREATIVE` if novelty is the dominant deficit
- `INSUFFICIENT` → `EVIDENCE` (default), MAY use `SIMULATION` if simulation_required dominates, MAY use `PARTNER_DEV` if adapter/pack repair is needed
- `UNKNOWN_SYN` → `PARTNER_DEV` or `EVIDENCE`, MAY use `FORMALIZE` if logic conflicts dominate

The agent SHOULD treat `recommended_protocol` in SideRecord as the system default and override only when user profile or local context justifies.

### E.5.2 Override contract
When the agent overrides `recommended_protocol`, it MUST emit an **AgentOverrideEvent**:
- `side_record_id`
- `from_protocol`
- `to_protocol`
- `override_reason` (short, specific)
- `timestamp_utc`

---

## E.6 Bank operations (HB/EB/GB)

### E.6.1 Storage (mandatory)
The working environment MUST store each SideRecord into the **Hypothesis Bank (HB)** (Appendix D).

The agent MAY add:
- `H_rank` adjustments (user-specific)
- `validity_scope` tags
- `supporting_refs[]` / `counter_refs[]` as they become available

### E.6.2 Error Bank (EB) view
The agent MUST treat EB as a **view** over HB (negative hypotheses with counter artifacts).  
An EB entry MUST remain reopenable.

### E.6.3 Gap Bank (GB)
For `INSUFFICIENT`, the agent SHOULD store recurring patterns into GB:
- missing provenance fields
- repeated low w_indep patterns
- repeated coverage slices absent
- simulation-required triggers

GB entries SHOULD include a reusable remediation checklist.

### E.6.4 Energy-saving rule
Before requesting expensive steps (new evidence crawl, simulation), the agent SHOULD:
1) query EB/HB for near-identical patterns and known counters,
2) query GB for standard remediation plans.

If a high-confidence match exists, the agent MAY short-circuit and propose the remediation plan.

---

## E.7 Human challenge loop (behavioral requirement)

If a human challenges any record (TRUE, rejected, insufficient, unknown):
- the agent MUST respond **non-dismissively** and explicitly acknowledge the objection,
- the agent MUST generate a re-evaluation plan (`AgentActionPlan`) linked to the challenge,
- the plan MUST specify expected artifacts (evidence, simulation artifacts, formalization) and the reopen trigger used (`HUMAN_CHALLENGE`).

---

## E.8 AgentActionPlan (minimum structure)

An action plan MUST include:
- `run_id`
- `target_side_record_ids[]`
- `selected_protocol`
- `next_actions[]` (ordered, each with expected output artifact)
- `stop_condition` (when to stop and re-run STEP 4/5/6)
- `risk_notes` (e.g., bias, overfitting to user preferences)

---

## E.9 Privacy & personalization guardrails

- `user_profile_ref` MUST be a stable pointer; raw PII MUST NOT be embedded in SideRecords.
- The agent MAY personalize protocol selection, but MUST NOT change:
  - step thresholds,
  - policy strictness,
  - coupling graph bindings,
  unless the user explicitly switches policy pack / config (logged event).

---

## E.10 Interop checklist (what to link)

- Appendix D: SideRecord format + banks semantics (mandatory)
- Appendix B: gap codes (mandatory for `INSUFFICIENT`)
- Appendix C: adapter provenance fields (source_id/cluster_id/cluster_size/w_indep)
- Protocol S: simulation evidence artifacts
- Protocol U: formalization/orphan consensus artifacts
