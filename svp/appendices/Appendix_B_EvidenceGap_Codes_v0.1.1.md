# Appendix B — EvidenceGap Codes (Normative Reference)
# Scope: STEP 5 v1.2 (Proof / Evidence Builder) → compatibility with STEP 6 v1.2 (SVP-EVAL)
# Rule: If a BLOCKER gap exists, STEP 5 MUST NOT emit an "OK" PrecheckReport.
# Revision: v0.1.1 (2026-02-13) — added provenance_missing_fields; aligned robustness wording to INSUFFICIENT

appendix_b_evidencegap_codes_v0_1:

  meta:
    status: "normative"
    applies_to_steps: ["STEP_5_v1.2", "STEP_5_v1.2.2"]
    downstream_dependency: "STEP_6_v1.2"
    default_severity_policy:
      BLOCKER: "halts emission to STEP 6; verdict must not be computed"
      WARN: "may proceed but caps confidence or forces downgrade to 0 on affected elements"
      INFO: "recorded for traceability only"

  codes:

    # ------------------------
    # Normalization / Schema
    # ------------------------
    - code: "normalization_missing_fields"
      severity: "BLOCKER"
      description: "Evidence line cannot be expressed as ElementEvidenceRecord with required fields."
      triggers:
        - "Any EvidenceSet record missing: evidence_class, element, sign, magnitude, credibility, reproducibility, independence, refs"
        - "Any E_sim-derived record missing robustness.verdict"
      required_fields:
        base: ["evidence_class","element","sign","magnitude","credibility","reproducibility","independence","refs"]
        if_e_sim: ["robustness.verdict"]
      downstream_impact: "STEP 6 cannot compute strength_score; must return UNKNOWN_SYN."
      typical_next_step: ["FIX_FIELDS","ADD_ADAPTER","RERUN_ARTIFACT"]
      notes: ["Adapters are allowed only if mapping is deterministic and lossless for required fields."]


    - code: "provenance_missing_fields"
      severity: "BLOCKER"
      description: "Provenance identity fields missing or non-deterministic; cluster independence cannot be computed safely."
      triggers:
        - "Any EvidenceSet record missing provenance.source_id"
        - "Any EvidenceSet record missing provenance.cluster_id"
        - "Adapter cannot derive stable source_id/cluster_id without guessing"
      required_fields:
        provenance: ["provenance.source_id","provenance.cluster_id"]
      downstream_impact: "STEP 6 MUST exclude affected records from ESS independence computation; if exclusion removes all records for an element, that element delta is capped to 0 and may force UNKNOWN_SYN."
      typical_next_step: ["ADD_SOURCE_METADATA","FIX_ADAPTER_RULES","RERUN_ADAPTER"]
      notes: ["cluster_size and w_indep are post-consolidation; missing source_id/cluster_id requires new metadata or a deterministic adapter rule."]
    - code: "evidence_normalization_incomplete"
      severity: "BLOCKER"
      description: "Normalization step failed globally; artifacts are not convertible to records."
      triggers:
        - "No deterministic adapter exists for one or more artifact classes"
        - "Mixed schemas within EvidenceSet without adapter declarations"
      downstream_impact: "STEP 6 must output UNKNOWN_SYN('evidence_normalization_incomplete')."
      typical_next_step: ["ADD_ADAPTER","STANDARDIZE_ARTIFACTS"]

    - code: "schema_version_mismatch"
      severity: "BLOCKER"
      description: "EvidenceSet/ProofPlan schema version not compatible with STEP 6 v1.2."
      triggers:
        - "EvidenceSet.version != 1.2"
        - "Record fields use deprecated names without adapter"
      downstream_impact: "Potential silent mis-aggregation."
      typical_next_step: ["MIGRATE_SCHEMA","ADD_ADAPTER"]

    # ------------------------
    # Scope / Coupling Graph
    # ------------------------
    - code: "missing_scope"
      severity: "BLOCKER"
      description: "U-SCOPE missing or not bound to evidence artifacts."
      triggers:
        - "scope_ref absent"
        - "scope_fit missing in evidence"
      downstream_impact: "Out-of-scope evidence cannot be filtered; No-Guessing."
      typical_next_step: ["REQUEST_SCOPE","RUN_PROTOCOL_U"]

    - code: "missing_terms"
      severity: "BLOCKER"
      description: "U-TERMS missing or essential terms undefined."
      triggers:
        - "terms_ref absent"
        - "undefined_terms contains essential terms"
      downstream_impact: "Cannot operationalize metrics; invalid falsifiers."
      typical_next_step: ["REQUEST_DEFS","RUN_PROTOCOL_U"]

    - code: "coupling_graph_unbound"
      severity: "BLOCKER"
      description: "No coupling graph binding; flow-affecting cannot be computed deterministically."
      triggers:
        - "coupling_graph_version missing"
      downstream_impact: "Routing to simulation becomes heuristic; unsafe."
      typical_next_step: ["BIND_GRAPH","SET_GRAPH_HASH"]

    - code: "flow_affecting_undetermined"
      severity: "BLOCKER"
      description: "claim_flow_affecting cannot be computed or is contradictory."
      triggers:
        - "FA rule cannot be applied due to missing intervention/scope info"
        - "conflicting FA outcomes from different modules without resolution"
      downstream_impact: "May skip required E_sim; unsafe."
      typical_next_step: ["FIX_FA_INPUTS","NARROW_SCOPE","ADD_INTERVENTION_SET"]

    # ------------------------
    # Simulation requirements
    # ------------------------
    - code: "missing_opmap"
      severity: "BLOCKER"
      description: "U-OPMAP missing while simulation is required or planned."
      triggers:
        - "claim_flow_affecting=true AND opmap_ref missing"
        - "S planned but no pass_criteria_ref"
      downstream_impact: "S cannot compute scoreboard; ad-hoc metrics forbidden."
      typical_next_step: ["RUN_PROTOCOL_U","ADD_U_OPMAP"]

    - code: "simulation_required_missing"
      severity: "BLOCKER"
      description: "Flow-affecting claim but E_sim not planned or not present."
      triggers:
        - "claim_flow_affecting=true AND no E_sim in EvidenceSet AND no S plan"
      downstream_impact: "STEP 6 computability gate fails -> UNKNOWN_SYN."
      typical_next_step: ["PLAN_SIMULATION","RUN_PROTOCOL_S"]

    - code: "e_sim_contract_incomplete"
      severity: "BLOCKER"
      description: "E_sim missing minimal contract fields required for robustness and borrowed routing."
      triggers:
        - "robustness.verdict missing"
        - "stress_suite.pass_rate missing"
        - "builder_vs_critic_delta missing"
        - "critical_edges_used missing"
        - "eps_ab_delta missing"
      downstream_impact: "STEP 6 cannot apply robustness gates; must downgrade or UNKNOWN."
      typical_next_step: ["RERUN_SIMULATION","BIND_PROFILE","ENABLE_STRESS_SUITE","ENABLE_AB_RUNS"]

    - code: "profile_binding_missing"
      severity: "BLOCKER"
      description: "Simulation outputs not bound to a declared profile/policy."
      triggers:
        - "E_sim.profile_id/profile_version missing"
        - "thresholds/eps absent and not derivable"
      downstream_impact: "Cannot interpret pass_rate/ab_delta; unsafe."
      typical_next_step: ["BIND_PROFILE","RERUN_SIMULATION_WITH_PROFILE"]

    - code: "stress_coverage_incomplete"
      severity: "BLOCKER"
      description: "Stress suite does not cover required nodes/critical edges for flow-affecting scenario."
      triggers:
        - "node coverage missing (not all 5 elements)"
        - "critical edges not each covered at least once"
      downstream_impact: "Robustness verdict becomes INSUFFICIENT; Step 6 will downgrade to 0."
      typical_next_step: ["EXPAND_STRESS_SUITE","DERIVE_CRITICAL_EDGES","RERUN_SIMULATION"]

    - code: "ab_mirror_missing"
      severity: "BLOCKER"
      description: "Builder/Critic A/B runs missing for simulation evidence used for decisions."
      triggers:
        - "No S-RUN-B counterpart"
        - "builder_vs_critic_delta cannot be computed"
      downstream_impact: "Mirror-safe robustness cannot be asserted."
      typical_next_step: ["RUN_CRITIC_SIDE","ENABLE_AB_PROTOCOL"]

    - code: "robustness_inconclusive_on_key_elements"
      severity: "WARN"
      description: "Key element decisions rely on E_sim with robustness INSUFFICIENT."
      triggers:
        - "SOC or BIO non-zero sign expected but only INSUFFICIENT E_sim available"
      downstream_impact: "STEP 6 will downgrade those elements to 0 under No-Guessing."
      typical_next_step: ["IMPROVE_COVERAGE","RERUN_SIMULATION","ADD_SECONDARY_LINE"]

    # ------------------------
    # Secondary line (SOC/BIO)
    # ------------------------
    - code: "secondary_line_missing_plan"
      severity: "BLOCKER"
      description: "Medium/long horizon expects SOC/BIO non-zero but no secondary independent line planned."
      triggers:
        - "horizon_class in {medium,long} AND expected SOC/BIO sign != 0 AND secondary_line_plan.required=false/absent"
      downstream_impact: "STEP 6 will cap confidence to weak and downgrade sign to 0."
      typical_next_step: ["ADD_E_CASE_OR_E_OPS","ADD_REPLICATION","INCREASE_INDEPENDENCE"]

    - code: "secondary_line_missing_evidence"
      severity: "BLOCKER"
      description: "Secondary line planned but not provided."
      triggers:
        - "secondary_line_plan.required=true but no record satisfies independence_min"
      downstream_impact: "Same as above; cannot assert SOC/BIO non-zero."
      typical_next_step: ["COLLECT_SECONDARY_DATA","RUN_INDEPENDENT_METHOD","ADD_CODE_INDEPENDENCE"]

    - code: "secondary_line_insufficient_independence"
      severity: "WARN"
      description: "Secondary line exists but fails independence minimum."
      triggers:
        - "data_independent < 1 OR (method_independent < 1 AND code_independent < 1)"
      downstream_impact: "May still proceed but Step 6 likely downgrades."
      typical_next_step: ["IMPROVE_INDEPENDENCE","ANNOTATE_INDEPENDENCE","REPEAT_INDEPENDENT_RUN"]

    # ------------------------
    # Borrowed benefit / tradeoffs
    # ------------------------
    - code: "borrowed_flag_missing"
      severity: "WARN"
      description: "Flow-affecting simulation lacks borrowed_flag / debt channels reporting."
      triggers:
        - "E_sim present but borrowed_flag null and no spillover/borrowed block"
      downstream_impact: "Borrowed benefit may be undetected; Step 6 may force UNKNOWN conservatively."
      typical_next_step: ["ENABLE_BORROWED_DETECTOR","ADD_SPILLOVER_REPORTING"]

    - code: "borrowed_benefit_blocking_true"
      severity: "BLOCKER"
      description: "Borrowed benefit detected and tradeoffs policy not explicitly allowing it."
      triggers:
        - "borrowed_benefit=true AND tradeoffs_not_allowed_default"
      downstream_impact: "STEP 6 must not output TRUE_SYN; route to UNKNOWN + ScopeNarrowingProposal/Empirics."
      typical_next_step: ["SCOPE_NARROWING","DEFINE_TRADEOFF_POLICY","EMPIRICS_REQUIRED"]

    - code: "tradeoff_policy_missing"
      severity: "WARN"
      description: "No hard_constraints/tradeoff_allowed policy bound; default is conservative."
      triggers:
        - "policy binding absent"
      downstream_impact: "More UNKNOWN outcomes; reduced ability to accept controlled tradeoffs."
      typical_next_step: ["DEFINE_POLICY_PROFILE","BIND_POLICY"]

    # ------------------------
    # Consistency / contradictions
    # ------------------------
    - code: "contradictory_evidence_signs"
      severity: "WARN"
      description: "Strong plus and minus evidence on same element without explanation."
      triggers:
        - "S_plus and S_minus both high for element (heuristic) AND no scope regime split provided"
      downstream_impact: "Step 6 likely yields 0 or HYPOTHESIS."
      typical_next_step: ["SPLIT_CLAIM","ADD_SCOPE_REGIMES","RUN_STRESS_OR_REPLICATION"]

    - code: "scope_mismatch"
      severity: "BLOCKER"
      description: "Evidence is out of scope relative to U-SCOPE boundaries."
      triggers:
        - "scope_fit.in_scope=false for key records"
      downstream_impact: "Evidence must be excluded; may trigger missing evidence blockers."
      typical_next_step: ["NARROW_OR_ADJUST_SCOPE","COLLECT_IN_SCOPE_EVIDENCE"]

    # ------------------------
    # Integrity / auditability
    # ------------------------
    - code: "missing_hash_bindings"
      severity: "WARN"
      description: "Evidence lacks hashes for reproducibility (graph/config/code/intervention/stress suite)."
      triggers:
        - "E_sim.audit missing one or more hashes"
      downstream_impact: "Reproducibility/credibility scoring should be reduced."
      typical_next_step: ["EMIT_HASHES","RERUN_WITH_AUDIT_BINDINGS"]

    - code: "witness_attestation_missing"
      severity: "INFO"
      description: "No witness attestation for artifacts in high-risk contexts."
      triggers:
        - "risk_class=HIGH and no witness signature/attestation bundle"
      downstream_impact: "Governance may refuse deployment; does not block computation by itself."
      typical_next_step: ["REQUEST_WITNESS_ATTESTATION"]

  severity_to_step5_decision:
    BLOCKER: "Stop -> emit EvidenceGapPlan + PrecheckReport(status=BLOCKED_EVIDENCE_PLAN_INCOMPLETE)"
    WARN: "Proceed allowed -> must add note and expect STEP 6 downgrades"
    INFO: "Proceed allowed -> trace only"

# Appendix B (continued) — EvidenceGap Codes (Normative)
# Adds: (1) standardized 'next_step' enums, (2) mapping from gaps to STEP 5 actions,
# (3) minimal remediation checklists per category.

appendix_b_evidencegap_codes_v0_1_continued:

  next_step_enums:
    - "FIX_FIELDS"
    - "ADD_ADAPTER"
    - "MIGRATE_SCHEMA"
    - "REQUEST_SCOPE"
    - "REQUEST_DEFS"
    - "BIND_GRAPH"
    - "SET_GRAPH_HASH"
    - "FIX_FA_INPUTS"
    - "ADD_INTERVENTION_SET"
    - "PLAN_SIMULATION"
    - "RUN_PROTOCOL_S"
    - "BIND_PROFILE"
    - "RERUN_SIMULATION"
    - "ENABLE_STRESS_SUITE"
    - "ENABLE_AB_PROTOCOL"
    - "DERIVE_CRITICAL_EDGES"
    - "EXPAND_STRESS_SUITE"
    - "ADD_SECONDARY_LINE"
    - "COLLECT_SECONDARY_DATA"
    - "IMPROVE_INDEPENDENCE"
    - "ENABLE_BORROWED_DETECTOR"
    - "ENABLE_SPILLOVER_REPORTING"
    - "SCOPE_NARROWING"
    - "DEFINE_TRADEOFF_POLICY"
    - "EMPIRICS_REQUIRED"
    - "SPLIT_CLAIM"
    - "ADD_SCOPE_REGIMES"
    - "RUN_REPLICATION"
    - "EMIT_HASHES"
    - "REQUEST_WITNESS_ATTESTATION"

  gap_to_step5_actions:
    normalization_missing_fields:
      step5_actions:
        - "Locate source artifact(s) producing the record"
        - "Populate missing fields OR attach deterministic adapter"
        - "Re-run Precheck"
      must_fix_before_emit: true

    evidence_normalization_incomplete:
      step5_actions:
        - "Standardize artifact shapes"
        - "Register adapters for each legacy class"
        - "Produce EvidenceSet records with canonical schema"
      must_fix_before_emit: true

    schema_version_mismatch:
      step5_actions:
        - "Migrate EvidenceSet/ProofPlan to v1.2"
        - "If not possible, provide adapter mapping old field names → new"
      must_fix_before_emit: true

    missing_scope:
      step5_actions:
        - "Run Protocol U to create U-SCOPE"
        - "Bind scope_ref to all evidence artifacts"
      must_fix_before_emit: true

    missing_terms:
      step5_actions:
        - "Run Protocol U to define terms"
        - "Update U-OPMAP metrics/pass_criteria accordingly"
      must_fix_before_emit: true

    coupling_graph_unbound:
      step5_actions:
        - "Bind coupling_graph_version"
        - "If available, compute and attach coupling_graph_hash"
      must_fix_before_emit: true

    flow_affecting_undetermined:
      step5_actions:
        - "Supply intervention set or edge targets"
        - "Narrow scope so FA rules can evaluate deterministically"
      must_fix_before_emit: true

    missing_opmap:
      step5_actions:
        - "Run Protocol U to produce U-OPMAP"
        - "Ensure pass_criteria_ref is present for stress suite"
      must_fix_before_emit: true

    simulation_required_missing:
      step5_actions:
        - "Add S_SIMULATION to ProofPlan"
        - "Run Protocol S or schedule it as required"
      must_fix_before_emit: true

    e_sim_contract_incomplete:
      step5_actions:
        - "Bind profile_id/profile_version"
        - "Enable stress suite generation"
        - "Enable A/B mirror (builder + critic)"
        - "Emit robustness.verdict, ab_delta, pass_rate, critical_edges_used"
      must_fix_before_emit: true

    profile_binding_missing:
      step5_actions:
        - "Re-run simulation under explicit profile"
        - "Record thresholds and eps from profile"
      must_fix_before_emit: true

    stress_coverage_incomplete:
      step5_actions:
        - "Derive critical_edges_used via Coupling Graph rule"
        - "Expand stress suite to cover each critical edge + all nodes"
        - "Re-run simulation"
      must_fix_before_emit: true

    ab_mirror_missing:
      step5_actions:
        - "Run critic side"
        - "Compute builder_vs_critic_delta and include in E_sim"
      must_fix_before_emit: true

    robustness_inconclusive_on_key_elements:
      step5_actions:
        - "Increase cycles to min_cycles"
        - "Increase stress_suite coverage"
        - "Reduce overrides; improve reproducibility"
      must_fix_before_emit: false

    secondary_line_missing_plan:
      step5_actions:
        - "Add secondary_line_plan to ProofPlan"
        - "Specify acceptable classes and independence minima"
      must_fix_before_emit: true

    secondary_line_missing_evidence:
      step5_actions:
        - "Collect or run independent E_case/E_ops"
        - "Or run E_rep with independent data/method/code"
      must_fix_before_emit: true

    secondary_line_insufficient_independence:
      step5_actions:
        - "Increase data independence (new dataset/source)"
        - "Increase method/code independence (different pipeline/implementation)"
      must_fix_before_emit: false

    borrowed_flag_missing:
      step5_actions:
        - "Enable borrowed benefit detector in simulation"
        - "Emit spillover/debt channels explicitly"
      must_fix_before_emit: false

    borrowed_benefit_blocking_true:
      step5_actions:
        - "Emit ScopeNarrowingProposal stub"
        - "Route to empirical validation OR define tradeoff policy"
      must_fix_before_emit: true

    tradeoff_policy_missing:
      step5_actions:
        - "Bind policy profile that defines hard_constraints/tradeoff_allowed"
        - "If absent, accept conservative default and document it"
      must_fix_before_emit: false

    contradictory_evidence_signs:
      step5_actions:
        - "Split claim or add scope regimes"
        - "Add replication/stress to disambiguate"
      must_fix_before_emit: false

    scope_mismatch:
      step5_actions:
        - "Exclude out-of-scope records"
        - "Collect in-scope evidence"
        - "Or adjust scope via Protocol U (governance-controlled)"
      must_fix_before_emit: true

    missing_hash_bindings:
      step5_actions:
        - "Emit audit hashes in E_sim"
        - "Prefer re-run to guarantee reproducibility"
      must_fix_before_emit: false

    witness_attestation_missing:
      step5_actions:
        - "Request witness attestation bundle"
      must_fix_before_emit: false

  remediation_checklists:

    checklist_normalization:
      goal: "All records satisfy ElementEvidenceRecord contract."
      steps:
        - "Ensure sign ∈ {+,0,-} for every element finding used."
        - "Ensure magnitude ∈ [0..1] exists (normalize)."
        - "Ensure independence is integer tuple (method,data,code)."
        - "Ensure refs[] contains artifact ids."
        - "If E_sim: ensure robustness.verdict exists."

    checklist_simulation_contract:
      goal: "E_sim is usable by STEP 6 robustness gates."
      steps:
        - "Bind to profile_id/profile_version."
        - "Emit critical_edges_used."
        - "Emit stress_suite cases + pass_rate + case_results."
        - "Emit builder_vs_critic_delta and eps_ab_delta."
        - "Emit robustness.verdict."
        - "Emit borrowed_flag or borrowed/debt block."

    checklist_secondary_line:
      goal: "SOC/BIO non-zero is supportable in medium/long horizon."
      steps:
        - "Add E_case or E_ops with data_independent>=1."
        - "Add method_independent>=1 OR code_independent>=1."
        - "Ensure sign direction aligns with primary."
        - "Bind to scope_fit and provide refs."

    checklist_coupling_binding:
      goal: "Flow-affecting decisions are deterministic."
      steps:
        - "Bind coupling_graph_version."
        - "Attach coupling_graph_hash when available."
        - "Derive critical_edges_used by rule."
        - "Record intervention_set_ref if applicable."


# Appendix B (final) — EvidenceGap Codes: severity escalation rules and auto-generated messages
# Adds: (1) escalation/downgrade logic, (2) "user-facing" gap summaries (still normative),
# (3) a compact lookup table for quick implementation.

appendix_b_evidencegap_codes_v0_1_final:

  severity_escalation_rules:
    # These rules prevent "papering over" missing requirements by mislabeling them WARN/INFO.
    - rule_id: "ESC-1"
      when: "claim_flow_affecting=true AND simulation_required_missing"
      then: {force_severity: "BLOCKER", reason: "Step 6 computability gate will fail."}

    - rule_id: "ESC-2"
      when: "e_sim_contract_incomplete"
      then: {force_severity: "BLOCKER", reason: "Robustness gates and borrowed routing are undefined."}

    - rule_id: "ESC-3"
      when: "horizon_class in {medium,long} AND element in {SOC,BIO} AND (secondary_line_missing_plan OR secondary_line_missing_evidence)"
      then: {force_severity: "BLOCKER", reason: "Step 6 must downgrade SOC/BIO to 0; non-zero cannot be asserted."}

    - rule_id: "ESC-4"
      when: "scope_mismatch affects any record with (evidence_class in {E_rep,E_fail,E_case,E_ops}) AND sign != 0"
      then: {force_severity: "BLOCKER", reason: "Out-of-scope strong evidence cannot be used safely."}

    - rule_id: "ESC-5"
      when: "normalization_missing_fields affects any record used for SOC/BIO"
      then: {force_severity: "BLOCKER", reason: "Key elements require strict aggregation contract."}

  warn_to_blocker_upgrade_triggers:
    # If these WARN gaps co-occur, they become BLOCKER because the combined uncertainty breaks safety.
    - trigger_id: "WB-1"
      when_all:
        - "borrowed_flag_missing"
        - "tradeoff_policy_missing"
      upgrade_to: "BLOCKER"
      reason: "Borrowed benefit may exist but cannot be evaluated; must not proceed."

    - trigger_id: "WB-2"
      when_all:
        - "missing_hash_bindings"
        - "profile_binding_missing"
      upgrade_to: "BLOCKER"
      reason: "Simulation not reproducible and not interpretable; cannot trust E_sim."

    - trigger_id: "WB-3"
      when_all:
        - "contradictory_evidence_signs"
        - "flow_affecting_undetermined"
      upgrade_to: "BLOCKER"
      reason: "Cannot disambiguate regime/flow; requires claim split or scope regimes."

  user_facing_gap_summaries:
    # These are concise standardized messages STEP 5 may attach into EvidenceGapPlan.reason.
    normalization_missing_fields: "Evidence records are missing required fields for Step 6 aggregation (magnitude/independence/refs)."
    simulation_required_missing: "Claim is flow-affecting but no simulation evidence is planned/present; Step 6 must return UNKNOWN."
    e_sim_contract_incomplete: "Simulation report lacks required robustness/stress/A-B delta fields; cannot be used."
    secondary_line_missing_plan: "Medium/long horizon requires a second independent line for SOC/BIO; it is not planned."
    secondary_line_missing_evidence: "Second independent SOC/BIO evidence line is required but absent."
    scope_mismatch: "Key evidence is out of scope; it must be excluded or scope must be revised."
    borrowed_benefit_blocking_true: "Borrowed benefit detected; without explicit tradeoff policy, TRUE is forbidden."

  compact_lookup_table:
    # Implementers can load this table for quick validations.
    # Fields: severity, must_fix_before_emit, typical_next_step (top 3), step6_effect
    - code: "normalization_missing_fields"
      severity: "BLOCKER"
      must_fix_before_emit: true
      typical_next_step: ["FIX_FIELDS","ADD_ADAPTER","RERUN_ARTIFACT"]
      step6_effect: "UNKNOWN_SYN (normalization incomplete)"

    - code: "simulation_required_missing"
      severity: "BLOCKER"
      must_fix_before_emit: true
      typical_next_step: ["PLAN_SIMULATION","RUN_PROTOCOL_S","BIND_PROFILE"]
      step6_effect: "UNKNOWN_SYN (missing coupling evidence)"

    - code: "e_sim_contract_incomplete"
      severity: "BLOCKER"
      must_fix_before_emit: true
      typical_next_step: ["ENABLE_STRESS_SUITE","ENABLE_AB_PROTOCOL","RERUN_SIMULATION"]
      step6_effect: "Robustness gates fail; downgrade/UNKNOWN"

    - code: "secondary_line_missing_plan"
      severity: "BLOCKER"
      must_fix_before_emit: true
      typical_next_step: ["ADD_SECONDARY_LINE","COLLECT_SECONDARY_DATA","RUN_REPLICATION"]
      step6_effect: "SOC/BIO forced to 0"

    - code: "borrowed_benefit_blocking_true"
      severity: "BLOCKER"
      must_fix_before_emit: true
      typical_next_step: ["SCOPE_NARROWING","DEFINE_TRADEOFF_POLICY","EMPIRICS_REQUIRED"]
      step6_effect: "TRUE forbidden; route UNKNOWN"

    - code: "tradeoff_policy_missing"
      severity: "WARN"
      must_fix_before_emit: false
      typical_next_step: ["DEFINE_TRADEOFF_POLICY","BIND_PROFILE","DOCUMENT_DEFAULT"]
      step6_effect: "More conservative UNKNOWN outcomes"

    - code: "missing_hash_bindings"
      severity: "WARN"
      must_fix_before_emit: false
      typical_next_step: ["EMIT_HASHES","RERUN_SIMULATION","BIND_PROFILE"]
      step6_effect: "Lower reproducibility -> lower strength"
