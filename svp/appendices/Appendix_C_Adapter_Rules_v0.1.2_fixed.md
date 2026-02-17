# Appendix C — Adapter Rules (Normative) v0.1.2 — FIXED

- YAML SHA256: `98c365cd0208e602c7ac2691607bef2fe4a0d6ae6cb78b25da5aeb4138568689`

```yaml
appendix_c_adapter_rules_v0_1_2:
  meta:
    status: normative
    version: 0.1.2
    applies_to_steps:
    - STEP_5_v1.2
    downstream: STEP_6_v1.2
    canonical_output: ElementEvidenceRecord
    adapter_registration_required: true
  global_constraints:
  - id: AR-0
    rule: 'Adapters must be pure functions: same input artifact → same output records.'
  - id: AR-1
    rule: Adapters must not upgrade evidence class (e.g., E_sim → E_rep is forbidden).
  - id: AR-2
    rule: Adapters must not create non-zero sign if input has no directional claim.
  - id: AR-3
    rule: If magnitude cannot be derived deterministically, set it to null and emit BLOCKER gap.
  - id: AR-4
    rule: If independence is boolean only, map to ints using AR-IND-1; else emit gap if ambiguous.
  - id: AR-5
    rule: If E_sim lacks robustness/stress/A-B, adapter must mark E_sim unusable → BLOCKER gap.
  - id: AR-6
    rule: All adapter outputs must include refs[] pointing to original artifact_id + adapter_id.
  - id: AR-PROV-0
    rule: All adapter outputs MUST include provenance.{source_id,cluster_id}; cluster_size and w_indep
      MUST be filled by STEP 5 consolidation before STEP 6.
  - id: AR-PROV-1
    rule: Adapters MUST derive a stable source_id deterministically (see provenance_rules). If not possible,
      emit BLOCKER normalization_missing_fields.
  - id: AR-PROV-2
    rule: Adapters MUST assign cluster_id deterministically (see provenance_rules). If not possible, emit
      BLOCKER normalization_missing_fields.
  - id: AR-PROV-3
    rule: STEP 5 consolidation MUST compute cluster_size and w_indep across the full EvidenceSet using
      policy cluster_alpha; missing → BLOCKER normalization_missing_fields.
  canonical_schema:
    fields_required: &id001
    - record_id
    - evidence_class
    - element
    - sign
    - magnitude
    - credibility
    - reproducibility
    - independence.method_independent
    - independence.data_independent
    - independence.code_independent
    - refs
    - provenance.source_id
    - provenance.cluster_id
    - provenance.cluster_size
    - provenance.w_indep
    required_if_e_sim:
    - robustness.verdict
    fields_required_pre_consolidation:
    - record_id
    - evidence_class
    - element
    - sign
    - magnitude
    - credibility
    - reproducibility
    - independence.method_independent
    - independence.data_independent
    - independence.code_independent
    - refs
    - provenance.source_id
    - provenance.cluster_id
    fields_required_post_consolidation: *id001
    notes:
    - fields_required applies to the post-consolidation EvidenceSet emitted by STEP 5.
    - Adapters MUST at minimum produce fields_required_pre_consolidation; cluster_size and w_indep are
      filled during STEP 5 batch consolidation.
  provenance_rules:
  - id: AR-PROV-SRC-1
    input_pattern: artifact has any stable source pointer (doi|url|dataset_id|artifact_id)
    mapping:
      source_id: derive_source_id(input)
      parent_source_id: input.parent_source_id | null
      dataset_id: input.dataset_id | null
      lab_id: input.lab_id | null
      publisher_id: input.publisher_id | null
    deterministic_derivation:
      normalized_doi:
      - 'strip prefixes: ''https://doi.org/'', ''doi:'''
      - trim whitespace; lowercase
      - 'validate pattern: startswith ''10.'' else treat as missing'
      canonical_url:
      - parse URL; lowercase scheme+host; drop default ports (80/443)
      - remove fragment (#...)
      - 'sort query params lexicographically; remove tracking params: utm_*, gclid, fbclid'
      - 'normalize path: collapse //, remove trailing ''/'' unless root'
      - re-emit as RFC3986 string
      artifact_fallback:
      - require source_artifact_pointer or artifact_id; else BLOCKER
      - 'format: artifact:<artifact_id>'
    required_side_effect:
      on_fail_emit_gap:
        code: normalization_missing_fields
        severity: BLOCKER
        reason: provenance.source_id cannot be derived deterministically
  - id: AR-PROV-CL-1
    input_pattern: derive cluster_id from policy-defined cluster_key_fields
    mapping:
      cluster_key: build_cluster_key(cluster_key_fields, input)
      cluster_id: cl:sha256(cluster_key)
    deterministic_derivation:
      build_cluster_key:
      - collect name=value for each field in cluster_key_fields if value is not null/empty
      - if result empty → cluster_key = 'source_id=' + source_id
      - join parts with '|'
      sha256:
      - hex digest over UTF-8 bytes of cluster_key
    required_side_effect:
      on_fail_emit_gap:
        code: normalization_missing_fields
        severity: BLOCKER
        reason: provenance.cluster_id cannot be derived deterministically
  batch_consolidation_rules:
  - id: AR-BATCH-PROV-1
    rule: After ALL adapters run, STEP 5 MUST compute cluster_size and w_indep for every record.
    algorithm:
      cluster_size: count_unique(source_id) within each cluster_id across the full EvidenceSet
      w_indep: round_half_up( 1 / (cluster_size ^ cluster_alpha), 6 )
      cluster_size_definition: cluster_size := count_unique(source_id) within each cluster_id across the
        full EvidenceSet (unique provenance sources).
      rounding: round_half_up(value, 6)
      round_half_up_definition: 'Deterministic rounding: half away from zero to p decimals. For x: round_half_up(x,p)=sign(x)*floor(abs(x)*10^p
        + 0.5)/10^p.'
    policy_binding:
      requires:
      - canonical_primitives.provenance.cluster_alpha
    on_fail_emit_gap:
      code: normalization_missing_fields
      severity: BLOCKER
      reason: cluster_size/w_indep cannot be computed without policy binding
  independence_rules:
  - id: AR-IND-1
    input_pattern: 'independence: true|false (single boolean)'
    mapping:
      true:
        method_independent: 0
        data_independent: 0
        code_independent: 0
        note: Boolean 'true' means 'unspecified independence present' is ambiguous; default to 0s.
      false:
        method_independent: 0
        data_independent: 0
        code_independent: 0
    required_side_effect:
      emit_gap:
        code: normalization_missing_fields
        severity: BLOCKER
        reason: Boolean independence is not sufficient for STEP 6 strength scoring; annotate independence
          tuple.
    notes:
    - We refuse to guess which independence dimension holds.
  - id: AR-IND-2
    input_pattern: 'independence: {independent: true, level: N} (legacy)'
    mapping:
      method_independent: '0'
      data_independent: N
      code_independent: '0'
    constraints:
    - N must be integer >=0 else BLOCKER
  magnitude_rules:
  - id: AR-MAG-1
    input_pattern: sign/mark present, magnitude absent
    mapping:
      magnitude: null
    required_side_effect:
      emit_gap:
        code: normalization_missing_fields
        severity: BLOCKER
        reason: Magnitude is required for STEP 6; cannot be inferred safely.
    allowed_fix:
    - Provide normalization method and raw metric or effect size.
  - id: AR-MAG-2
    input_pattern: raw_metric + normalization available in artifact
    mapping:
      magnitude: normalize(raw_metric, normalization_spec)
    constraints:
    - normalization_spec must be explicit and deterministic
    - normalization must clip to [0..1]
    notes:
    - If normalization_spec missing → AR-MAG-1.
  sign_rules:
  - id: AR-SIGN-1
    input_pattern: legacy mark in {+,-,0} (string)
    mapping:
      sign: same
  - id: AR-SIGN-2
    input_pattern: legacy mark in {PLUS, MINUS, NEUTRAL}
    mapping:
      PLUS: +
      MINUS: '-'
      NEUTRAL: '0'
  - id: AR-SIGN-3
    input_pattern: effect described only in prose without explicit direction
    mapping:
      sign: '0'
    required_side_effect:
      emit_gap:
        code: normalization_missing_fields
        severity: BLOCKER
        reason: Direction not explicit; cannot convert prose to sign without guessing.
  evidence_class_rules:
  - id: AR-CLASS-1
    input_pattern: artifact class string
    mapping:
      S_REPORT: E_sim
      SIM_REPORT: E_sim
      OPS_REPORT: E_ops
      CASE_REPORT: E_case
      REPLICATION_REPORT: E_rep
      FAIL_REPORT: E_fail
      DERIVATION_NOTE: E_log
    constraints:
    - Unknown class => BLOCKER gap 'evidence_normalization_incomplete'
  robustness_rules:
  - id: AR-ROB-1
    input_pattern: E_sim without robustness.verdict
    mapping:
      robustness: null
    required_side_effect:
      emit_gap:
        code: e_sim_contract_incomplete
        severity: BLOCKER
        reason: E_sim lacks robustness; STEP 6 cannot apply robustness gates.
  - id: AR-ROB-2
    input_pattern: has stress_pass_rate and ab_delta and thresholds
    mapping:
      robustness.verdict: compute_verdict(stress_pass_rate, ab_delta, min_stress_pass, eps_ab_delta)
    compute_verdict:
      if:
      - condition: stress_pass_rate >= min_stress_pass AND ab_delta <= eps_ab_delta
        verdict: STABLE
      - condition: stress_pass_rate < min_stress_pass OR ab_delta > eps_ab_delta
        verdict: UNSTABLE
      else: INSUFFICIENT
    constraints:
    - min_stress_pass and eps_ab_delta must be bound to a profile; else BLOCKER profile_binding_missing
  borrowed_rules:
  - id: AR-BOR-1
    input_pattern: E_sim has borrowed_benefit block
    mapping:
      borrowed_flag: borrowed_benefit.is_borrowed
    notes:
    - If borrowed_benefit missing, borrowed_flag may be null (WARN) unless flow-affecting strict mode
      requires it.
  - id: AR-BOR-2
    input_pattern: E_sim has spillover paths but no borrowed flag
    mapping:
      borrowed_flag: null
    required_side_effect:
      emit_gap:
        code: borrowed_flag_missing
        severity: WARN
        reason: Spillover present but borrowed evaluation not explicit.
  element_extraction_rules:
  - id: AR-ELEM-1
    input_pattern: artifact has element_findings dict
    mapping:
      produce_records_for_each_element:
        source: element_findings
        fields:
          sign: element_findings.<e>.sign
          magnitude: element_findings.<e>.magnitude
    constraints:
    - Missing element entry is allowed; record omitted for that element.
  - id: AR-ELEM-2
    input_pattern: artifact states affected_elements list + single sign/magnitude
    mapping:
      produce_records_for_each_element:
        source: affected_elements
        fields:
          sign: artifact.sign
          magnitude: artifact.magnitude
  record_synthesis:
  - id: AR-REC-1
    rule: Each produced record_id must be deterministic.
    method:
      record_id: hash(co_id + evidence_id + element + adapter_id) truncated
    refs_policy:
      refs:
      - evidence_id
      - adapter_id
      - source_artifact_pointer
  - id: AR-REC-2
    rule: Credibility/reproducibility defaults are forbidden.
    enforcement:
      if_missing: emit BLOCKER normalization_missing_fields
    notes:
    - These must be provided by evidence artifact or by policy-defined scoring module.
  adapter_bundles:
  - bundle_id: BUNDLE_LEGACY_MARKS_ONLY
    intent: Legacy artifacts with marks but no magnitude/independence tuple
    applies_to:
    - E_log
    - E_case
    - E_ops
    uses_rules:
    - AR-PROV-SRC-1
    - AR-PROV-CL-1
    - AR-SIGN-1
    - AR-MAG-1
    - AR-IND-1
    - AR-REC-1
    - AR-REC-2
    expected_outcome: BLOCKER until magnitude/independence annotated
  - bundle_id: BUNDLE_SIM_REPORT_MINIMAL
    intent: Simulation report missing explicit robustness but has pass_rate and ab_delta
    applies_to:
    - E_sim
    uses_rules:
    - AR-PROV-SRC-1
    - AR-PROV-CL-1
    - AR-CLASS-1
    - AR-ROB-2
    - AR-BOR-1
    - AR-ELEM-1
    - AR-REC-1
    - AR-REC-2
    expected_outcome: Produces usable E_sim records if profile binding exists
  - bundle_id: BUNDLE_PROSE_ONLY
    intent: Free-form prose evidence
    applies_to:
    - unknown
    uses_rules:
    - AR-PROV-SRC-1
    - AR-PROV-CL-1
    - AR-SIGN-3
    - AR-MAG-1
    - AR-IND-1
    expected_outcome: Always BLOCKER (cannot infer direction/magnitude safely)
  concrete_adapters:
  - adapter_id: ADP_LEGACY_SVP_MARKS_V0
    status: normative
    input_family: SVP-style legacy mark tables
    detects:
      required_fields_any:
      - element_marks
      - marks
      - ElementFindings
    mapping_pipeline:
    - step: class_map
      rule: AR-CLASS-1
    - step: provenance
      rules:
      - AR-PROV-SRC-1
      - AR-PROV-CL-1
    - step: element_extract
      rule: AR-ELEM-1
    - step: sign_map
      rule: AR-SIGN-1
    - step: magnitude
      rule: AR-MAG-1
    - step: independence
      rule: AR-IND-1
    - step: synthesis
      rules:
      - AR-REC-1
      - AR-REC-2
    emits_gaps:
    - code: normalization_missing_fields
      severity: BLOCKER
      when: magnitude missing OR independence tuple missing
    notes:
    - 'This adapter forces modernization: marks alone are insufficient for STEP 6 v1.2.'
  - adapter_id: ADP_SIM_REPORT_NO_ROBUSTNESS
    status: normative
    input_family: Simulation report with pass_rate + ab_delta but no robustness.verdict
    detects:
      required_fields_all:
      - stress_suite.pass_rate
      - builder_vs_critic_delta
      - eps_ab_delta
    mapping_pipeline:
    - step: class_map
      rule: AR-CLASS-1
    - step: provenance
      rules:
      - AR-PROV-SRC-1
      - AR-PROV-CL-1
    - step: profile_binding_check
      on_fail_emit_gap:
        code: profile_binding_missing
        severity: BLOCKER
        reason: Cannot compute robustness verdict without thresholds.
    - step: robustness_compute
      rule: AR-ROB-2
    - step: borrowed
      rule: AR-BOR-1
    - step: element_extract
      rule: AR-ELEM-1
    - step: sign_map
      rule: AR-SIGN-1
    - step: magnitude
      rule: AR-MAG-2
    - step: independence
      rule: AR-IND-2
      fallback: AR-IND-1
    - step: synthesis
      rules:
      - AR-REC-1
      - AR-REC-2
    emits_gaps:
    - code: e_sim_contract_incomplete
      severity: BLOCKER
      when: stress suite missing OR ab_delta missing OR critical_edges_used missing
    - code: stress_coverage_incomplete
      severity: BLOCKER
      when: flow_affecting=true AND critical_edges_used not covered
    - code: borrowed_flag_missing
      severity: WARN
      when: borrowed_flag cannot be derived
    notes:
    - If critical_edges_used absent, adapter MUST emit BLOCKER e_sim_contract_incomplete.
  - adapter_id: ADP_OPS_METRICS_ONLY
    status: normative
    input_family: Ops telemetry with metrics but no element_findings
    detects:
      required_fields_all:
      - measurements
      - metric_id
      - value
    mapping_pipeline:
    - step: class_map
      map_to: E_ops
    - step: provenance
      rules:
      - AR-PROV-SRC-1
      - AR-PROV-CL-1
    - step: element_inference
      mode: forbidden
      on_fail_emit_gap:
        code: normalization_missing_fields
        severity: BLOCKER
        reason: Cannot infer element from ops metrics without explicit mapping.
    - step: require_opmap_link
      on_fail_emit_gap:
        code: missing_opmap
        severity: BLOCKER
        reason: Need U-OPMAP mapping metric_id -> element/sign/magnitude.
    remediation:
      required: Add U-OPMAP mapping or explicit element_findings in ops artifact.
    notes:
    - This prevents 'metric worship' without semantic operationalization.
  - adapter_id: ADP_CASE_STUDY_PROSE_PLUS_OUTCOME
    status: normative
    input_family: Case study with outcomes table + narrative
    detects:
      required_fields_any:
      - outcomes
      - results
    mapping_pipeline:
    - step: class_map
      map_to: E_case
    - step: provenance
      rules:
      - AR-PROV-SRC-1
      - AR-PROV-CL-1
    - step: require_direction
      on_fail_emit_gap:
        code: normalization_missing_fields
        severity: BLOCKER
        reason: Case outcomes present but effect direction/sign not explicit.
    - step: magnitude_compute
      rule: AR-MAG-2
      require: normalization_spec
    - step: independence_check
      rule: AR-IND-2
      fallback: AR-IND-1
    - step: synthesis
      rules:
      - AR-REC-1
      - AR-REC-2
    notes:
    - Narrative is allowed, but direction+magnitude must be explicit or computable.
  - adapter_id: ADP_REPLICATION_TABLE
    status: normative
    input_family: Replication report with PASS/FAIL runs
    detects:
      required_fields_all:
      - replications
      - run_id
      - result
    mapping_pipeline:
    - step: class_map
      map_to: E_rep
    - step: provenance
      rules:
      - AR-PROV-SRC-1
      - AR-PROV-CL-1
    - step: derive_sign
      rule:
        method: pass_rate_to_sign
        pass_rate_thresholds:
          plus: 0.8
          minus: 0.2
        on_midrange: '0'
      constraints:
      - Thresholds must be declared in artifact or policy; else emit schema_version_mismatch (BLOCKER).
    - step: magnitude
      rule:
        method: magnitude = abs(pass_rate - 0.5) * 2
        clip: true
    - step: independence
      require_tuple: true
      on_fail_emit_gap:
        code: normalization_missing_fields
        severity: BLOCKER
        reason: Replication without independence tuple is unusable for STEP 6.
    - step: synthesis
      rules:
      - AR-REC-1
      - AR-REC-2
    emits_gaps:
    - code: contradictory_evidence_signs
      severity: WARN
      when: replications split across regimes without scope regimes defined
    notes:
    - E_rep is strong; therefore independence annotation is mandatory.
  forbidden_inference_rules:
  - id: FORBID-1
    forbid: Inferring element from metric names, variable names, or prose keywords.
    reason: Too easy to smuggle assumptions; must be explicit via U-OPMAP or element_findings.
  - id: FORBID-2
    forbid: Inferring sign from sentiment analysis of prose.
    reason: Non-deterministic and not evidence-based under this spec.
  adapter_conformance_checks:
  - check_id: C-1
    name: Determinism
    requirement: Given identical input artifact bytes + adapter_id, output records MUST be identical.
    failure_gap:
      code: evidence_normalization_incomplete
      severity: BLOCKER
  - check_id: C-2
    name: No evidence class upgrades
    requirement: Adapter MUST NOT map weaker evidence to stronger class.
    examples_forbidden:
    - E_sim -> E_rep
    - E_case -> E_rep
    failure_gap:
      code: schema_version_mismatch
      severity: BLOCKER
  - check_id: C-3
    name: No invented magnitude
    requirement: Magnitude may be computed only with explicit normalization_spec or raw metric mapping.
    failure_gap:
      code: normalization_missing_fields
      severity: BLOCKER
  - check_id: C-4
    name: No invented sign
    requirement: Sign must be explicit or computed from declared thresholds/rules.
    failure_gap:
      code: normalization_missing_fields
      severity: BLOCKER
  - check_id: C-5
    name: E_sim minimal contract
    requirement: If mapping to E_sim, robustness.verdict and stress_suite + ab_delta must exist or be
      computable from bound profile.
    failure_gap:
      code: e_sim_contract_incomplete
      severity: BLOCKER
  - check_id: C-6
    name: Independence tuple required
    requirement: Independence must be integer tuple; boolean-only independence is BLOCKER until annotated.
    failure_gap:
      code: normalization_missing_fields
      severity: BLOCKER
  - check_id: C-7
    name: Refs and provenance
    requirement: Each record MUST include refs[] with (source evidence_id, adapter_id).
    failure_gap:
      code: normalization_missing_fields
      severity: BLOCKER
  adapter_audit_logging_requirements:
    log_schema:
      log_id: ADAPTERLOG_0001
      adapter_id: ADP_...
      input_artifact_id: E_...
      input_hash: sha256:...
      output_records_hash: sha256:...
      produced_record_ids: []
      emitted_gaps: []
      timestamp_utc: YYYY-MM-DDThh:mm:ssZ
      notes: []
  minimal_test_vectors:
  - vector_id: TV-1
    name: Legacy mark-only artifact must BLOCK
    input:
      class: E_case
      element_findings:
        SOC:
          sign: +
          notes: improved
      quality:
        credibility: 0.8
        reproducibility: 0.4
        independence: true
    expected:
      produced_records: 0
      emitted_gaps:
      - code: normalization_missing_fields
        severity: BLOCKER
        must_include_missing_fields:
        - magnitude
        - independence.*
  - vector_id: TV-2
    name: E_sim with pass_rate + ab_delta + bound profile computes robustness
    input:
      class: SIM_REPORT
      profile_binding:
        profile_id: SCS_PROFILE_DEFAULT_v0.1
        thresholds:
          min_stress_pass: 0.9
        eps_ab_delta: 0.001
      stress_suite:
        pass_rate: 0.95
      builder_vs_critic_delta: 0.0005
      eps_ab_delta: 0.001
      critical_edges_used:
      - SOC->AI
      element_findings:
        AI:
          sign: +
          magnitude: 0.35
      quality:
        credibility: 0.7
        reproducibility: 0.6
        independence:
          method_independent: 1
          data_independent: 0
          code_independent: 1
    expected:
      produced_records: 1
      record_fields:
        evidence_class: E_sim
        element: AI
        sign: +
        magnitude: 0.35
        robustness.verdict: STABLE
      emitted_gaps: []
  - vector_id: TV-3
    name: Ops metrics without OPMAP must BLOCK (no element inference)
    input:
      class: OPS_REPORT
      measurements:
      - metric_id: latency_p95
        value: 120
    expected:
      produced_records: 0
      emitted_gaps:
      - code: missing_opmap
        severity: BLOCKER
  - vector_id: TV-4
    name: Replication table produces E_rep sign from declared thresholds
    input:
      class: REPLICATION_REPORT
      thresholds:
        plus: 0.8
        minus: 0.2
      replications:
      - run_id: r1
        result: PASS
      - run_id: r2
        result: PASS
      - run_id: r3
        result: PASS
      - run_id: r4
        result: FAIL
      element: AI
      independence:
        method_independent: 1
        data_independent: 2
        code_independent: 1
      quality:
        credibility: 0.9
        reproducibility: 0.8
    expected:
      produced_records: 1
      record_fields:
        evidence_class: E_rep
        sign: +
        magnitude_range:
        - 0.0
        - 1.0
      emitted_gaps: []
  compliance_outcome:
    compliant_if:
    - All adapter_conformance_checks pass
    - All required audit log fields emitted
    - Minimal test vectors pass
    non_compliant_if:
    - Any BLOCKER emitted due to adapter misbehavior (not due to missing input fields)
    - Any forbidden inference performed (e.g., element inference when mode=forbidden)
    - Any evidence_class upgrade detected (AR-1 violation)
    - Any missing required audit log field
  deterministic_primitives:
    round_half_up:
      definition: half away from zero to p decimals
      spec: round_half_up(x,p)=sign(x)*floor(abs(x)*10^p + 0.5)/10^p
      notes:
      - Used for w_indep rounding to avoid platform-dependent bankers rounding.
```
