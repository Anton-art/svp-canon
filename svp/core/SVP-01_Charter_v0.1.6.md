# SVP-01 Charter v0.1.6 — Syntropy Verdict Protocol (Charter)

**Status:** Charter (Semantics + Prohibitions + Human Role)  
**Executable spec (current):** SVP-EVAL Policy Pack v0.1 + Steps 1–6 + Modules A,B + Protocols S,U (see §6 Canon Set)  
**Primary truth gating:** Element-5 (HUM/AI/BIO/TEC/SOC)  
**Axis-5 role:** secondary; downgrade-only (may narrow scope / downgrade TRUE→UNKNOWN; never upgrade UNKNOWN→TRUE)

---

## 0) Purpose

SVP-01 defines:

- **Meaning** of Truth/Lie/Hypothesis as **syntropy-truth** (impact across Element-5 inside a declared scope).
- **Prohibitions** to prevent “pretty logic” from being mistaken for truth.
- **Human Role Contract**: **witness-only** (no normative judging).
- **Simulation role**: **evidence-only** (never a truth canon).

SVP-01 is a **charter**, not a full algorithm. The computable pipeline is defined by **SVP-EVAL** (Policy Pack + Steps + Modules + Protocols).

---

## 1) Core Terms

### 1.1 Elements (Element-5)
`ELEM = { HUM, AI, BIO, TEC, SOC }`

A claim is evaluated as an **effect** (or expected effect) across these coupled elements, within scope constraints.

### 1.2 Claim Object (CO)
A **Claim Object** is a structured claim evaluated under:
- declared **domain** and **horizon**,
- explicit **scope boundaries**,
- a defined **evidence set**,
- and a **witness record** (provenance).

### 1.3 Verdicts (Syntropy Verdict)
`syntropy_verdict ∈ { TRUE_SYN, FALSE_SYN, UNKNOWN_SYN }`

- **TRUE_SYN:** evidence supports non-harmful (or net-positive) contribution to Element-5 inside scope.
- **FALSE_SYN:** evidence supports net-harm / contradiction with Element-5 inside scope.
- **UNKNOWN_SYN:** insufficient evidence or insufficient determinism (gaps) → fail-closed.

### 1.4 Status & failure codes
SVP-EVAL uses fail-closed statuses:
- `INSUFFICIENT` (replaces legacy inconclusive status)
- `BLOCKED_*` (policy missing, context missing, index unbound, etc.)

---

## 2) Truth semantics (syntropy-truth)

A claim may be “true” in SVP only relative to:
- **scope**, **time horizon**, **risk class**, and **evidence contract**.

Truth is not “coherence”; truth is **bounded evidence of beneficial coupling** (or at least non-harm) across Element-5.

---

## 3) Prohibitions (Hard rules)

The system MUST NOT treat the following as truth signals:

1) **Cohesion / elegance / compressibility** alone  
2) **Novelty** alone  
3) **High confidence language** alone  
4) **Single-source authority** without provenance independence  
5) **Simulation output** without provenance / coverage / adversarial stress disclosure  
6) **Human “judgement”** as a truth upgrade mechanism  

If any of these are the only “support”, verdict MUST be `UNKNOWN_SYN` and SVP-EVAL SHOULD emit `INSUFFICIENT` with gap codes.

---

## 4) Human Role Contract (Witness-only)

Humans MAY:
- attest provenance, scope boundaries, observed artifacts,
- provide domain constraints (what counts as harm in scope).

Humans MUST NOT:
- upgrade UNKNOWN→TRUE by “authority”,
- override fail-closed gaps without producing evidence artifacts.

All human inputs MUST be logged as **witness attestations**.

---

## 5) Simulation role (Evidence-only)

Simulation is a controlled generator of **E_sim** evidence:
- It can improve robustness assessment (Module B RTT).
- It can strengthen evidence weighting (Module A ESS).
- It cannot define truth canonically.

If simulation is required but missing/insufficient → `INSUFFICIENT` + gap codes.

---

## 6) Canon Set (Release snapshot)

**Snapshot date:** 2026-02-13

### 6.0 Pipeline versions (canonical)
- STEP 1 v1.1.2
- STEP 2 v1.1.2
- STEP 3 v1.1.2
- STEP 4 v1.1.4
- STEP 5 v1.2.2
- STEP 6 v1.2.3

- Module A (ESS) v0.1.2
- Module B (RTT) v0.1.2
  
**Format rule:** when there is a choice, canon is **MD/YAML/JSON**; DOCX is treated as source-only / legacy.

### 6.1 Canonical artifacts (file + SHA256)
| # | File | SHA256 |
|---:|---|---|
| 1 | `SVP_v0.2.3.md` | `c69e4254bf0300e455d1e77cdb8f5ff92583acd8359377a41b1da11b54091470` |
| 2 | `SVP-02_v0.2.3.yaml` | `05021778960003dfe4488d28bdfee7055d8359cec0e9566485fef1b0dfbe1e08` |
| 3 | `SVP-EVAL_PolicyPack_v0.1.1_updated_fixed_step6.md` | `a9defeea95225de75026083fc759e141a8ebcf46c428555630809720df1103f6` |
| 4 | `SVP-EVAL_PolicyPack_v0.1.1_updated_fixed_step6.yaml` | `8608fcb068dedabc78cb4af2521714cdfcdf606648791385854014969a09b7ae` |
| 5 | `STEP_1_v1.1.3_Input_Intake_Claim_Extraction_Ultralight_Noise_Gate.md` | `320acb56d5cd5eec9669f273a736c1d2d63375b4c8d2249e668147be0baa3d58` |
| 6 | `STEP_2_v1.1.2_Stage_A_Ultracheap_Chaos_Score_no_models.md` | `8a5445c6e2137f7e907df48964e90bfeb82dc750bbb008ec9f3290642a3926e2` |
| 7 | `STEP_3_v1.1.2_ChaosFilter_Stage_B_Main_Chaos_Index.md` | `ea814259be5caa56e9d212505841089a00ad3198d505207351c65673c91b8656` |
| 8 | `STEP_4_v1.1.5_Novelty_Connectedness_Gate_fixed.md` | `bce1af007014b66a32e9177ab7878fe42cfb7c91001076f0cc2fac0656d77375` |
| 9 | `Appendix_A4_STEP4_YAML_Templates_v1.1.4.yaml` | `ed3c2f278b8f97e5f65b76e2996edf56d430d4dcecaa2e6bc23c86dcd4f5ddab` |
| 10 | `STEP_5_v1.2.2_Proof_Evidence_Builder_fixed.md` | `d1b6617c836bcf6dd827e6767902b658bc289612ebf7a2c4c7366e3264132bbb` |
| 11 | `Appendix_A_STEP5_YAML_Templates_v1.2.2_fixed.yaml` | `742636dabd32a25754bc75ef9cbf3fe31d6bdeb91b92ec92dd56d5f03eb30eb4` |
| 12 | `Appendix_B_EvidenceGap_Codes_v0.1.1.md` | `fdd2ee727049e20551bb166464cc0e12b23c8876b8d6d6eab27e3569ce5b8d42` |
| 13 | `Appendix_C_Adapter_Rules_v0.1.2_fixed.md` | `64cb347868ac1a2abd25d91c1cfe21efa736029b0cccfa5636e72638b263baf0` |
| 14 | `Module_A_ESS_v0.1.2_fixed.md` | `ee842892f6440600f88a83e6d8343c09c6806b449d37036ad6ba999920778594` |
| 15 | `Module_B_RTT_v0.1.2_fixed.md` | `b7591c97133bd8df977ded4caf178a8d01ce5c1084cdc4abab2dc3b40a18bc5d` |
| 16 | `YAML_Templates_Modules_A_B_v0.1.2.yaml` | `edfcb9b862398592283cff5200c56a1216a2dda46a8e79dc6cc1758a253cec93` |
| 17 | `Protocol_S_v1.0.2_Simulation_Evidence_Protocol.md` | `bf5c3d7574cad6229584ebe3c34ffb1220f553dfe250516d3a37e2c204396c94` |
| 18 | `Simulation_Scenario_Spec_v0.1.1.yaml` | `29b23627eff85ce4674b78a9b6ee1e58e9a12952f983aea44e332e7a8b5be776` |
| 19 | `e_sim.schema.min.v0.1.1.json` | `51a09ef5e72e91146e38fd4d8c93df8985e2b8f93fbe87ad079acea48494767d` |
| 20 | `SCS_Coupling_Graph_v0.1.2.yaml` | `8b54815017dd4005e930a05d623560f9a94a43c13cab9fc2f2bb0ac67018ca3c` |
| 21 | `Protocol_U_v1.0.2_Formalization_Orphan_Consensus.md` | `1ee456b1670b2a200e4d346497914a73d2dd9fd1351eadaea14177ca4b51d205` |
| 22 | `STEP_6_v1.2.3_SVP-EVAL_fixed.md` | `7436e133f4d0049f3ec4d684dc3ca2ececd9cc284f29ce3adcdcf29dac456b26` |
### 6.2 Canon boundary
The following legacy documents are **NOT canonical** until re-issued in MD with explicit version bump and hash:
- `SVP v0.2.2.docx`
- `SVP-02 v0.2.2 (YAML).docx`
- any other DOCX not listed in §6.1.

---

## 7) Release discipline (Normative)

A release MUST include:
- updated canon files (MD/YAML/JSON),
- a manifest of SHA256 hashes,
- policy required_keys coverage,
- EvidenceGap registry coverage (Appendix B),
- and a minimal “golden cases” smoke test set.

Failing any of the above → release is blocked (`BLOCKED_RELEASE_INTEGRITY`).
