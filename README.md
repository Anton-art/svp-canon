# SVP — Canon Repo (v0.2.11)

This repository is the **canonical specification set** for the SVP filtering pipeline and SVP-EVAL: Steps 1–6, appendix contracts, metric modules, simulation/formalization protocols, and a unified **PolicyPack** (single YAML configuration).

Goal: enable an engineer/researcher to implement the pipeline **in the real world** (external indices, retrieval, graphs, simulations, hypothesis/error banks, agent feedback loop) and produce **reproducible** verdicts and artifacts.

---

## 1) What SVP is (in 12 lines)

SVP is a pipeline of **six gate steps** that filters incoming ideas (human or agent output), removes noise/chaos, checks novelty and connectedness, builds evidence, and produces a final **SVP-EVAL verdict** (TRUE / INSUFFICIENT / UNKNOWN…).

Key engineering principle: **nothing discarded is wasted**. At every step, the system emits **SideRecords** (side outputs) and updates three banks:
- **HB** — Hypothesis Bank (all candidates / hypotheses),
- **EB** — Error Bank (negative hypotheses / reusable failures; saves compute),
- **GB** — Gap Bank (recurring evidence gaps + closure plans).

A personal agent receives SideRecords and selects protocols (learning / creative / partner-developer / evidence / simulation / formalization). The human may challenge any outcome and trigger re-evaluation.

---

## 2) Recommended reading order

1) **Canon Index (registry + hashes):**  
   `svp/core/SVP_Canon_Index_v0.2.11.md`

2) **Pipeline steps:**  
   `svp/steps/STEP_1_...` → `STEP_6_...`

3) **Side outputs & banks:**  
   `svp/appendices/Appendix_D_SideOutputs_Banks_Contract_v0.1.md`

4) **Agent Feedback Loop:**  
   `svp/appendices/Appendix_E_Agent_Feedback_Loop_Contract_v0.1.md`

5) **PolicyPack (unified YAML):**  
   `svp/policy/SVP-EVAL_PolicyPack_v0.1.2_updated_fixed_step6.yaml`

6) **Simulation / formalization protocols:**  
   `svp/protocols/Protocol_S_...` and `svp/protocols/Protocol_U_...`

---

## 3) What is normative vs. explanatory

**Normative (required for compatibility):**
- Steps 1–6 (gate logic + input/output contracts)
- Appendix B/C/D/E (gap codes; adapter/provenance rules; SideRecord + banks; agent loop)
- SVP-EVAL PolicyPack (required keys + thresholds + fail-closed rules)
- MANIFEST / Canon Index (integrity, version pinning)

**Explanatory:**
- audit/consistency reports (CrossCheck, Patchlist)
- this README

---

## 4) Repository layout

- `svp/core/` — core docs + Canon Index  
- `svp/steps/` — Steps 1…6 (main pipeline)  
- `svp/appendices/` — contracts/templates (A/B/C/D/E)  
- `svp/modules/` — Module A (ESS), Module B (RTT), shared YAML templates  
- `svp/policy/` — SVP-EVAL PolicyPack (YAML + MD wrapper)  
- `svp/protocols/` — Protocol S (simulation), Protocol U (formalization)  
- `svp/simulation/` — schemas/scenarios/coupling graph  
- `svp/audit/` — CrossCheck, Patchlist, MANIFESTs, internal repo manifest

---

## 5) How SVP connects to the external world (tools & infrastructure)

SVP is designed so that expensive operations happen **only on explicit triggers** and remain reproducible via artifacts:

### Steps 1–3 (cheap filtering)
- tokenization/normalization/light heuristics
- optional lightweight retrieval for sanity checks (not proof)

### Step 4 (novelty/near-dup + connectedness)
- **MinHash/LSH + SimHash** for near-duplicate detection and clustering
- KGR / graph representations for connectedness
- optional: search engine / vector DB / graph DB

### Step 5 (evidence builder)
- retrieval over sources, provenance tracking, source clustering
- independence weight computation (`w_indep`) and gap handling

### Step 6 (SVP-EVAL)
- application of ESS/RTT, policy binding, fail-closed on missing requirements
- escalation to Protocol S/U when triggered

### Protocol S / Protocol U
- Protocol S: simulator/compute stack + artifact storage + reproducibility
- Protocol U: formal languages/schemas + consensus procedures

---

## 6) Side outputs: what happens to items that do not pass

Steps 2–6 **must emit** a SideRecord (Appendix D).  
A SideRecord returns to the working environment and to the agent with:
- status (`REJECTED_CHAOS`, `REJECTED_NOVELTY`, `INSUFFICIENT`, `UNKNOWN_SYN`…),
- `reason_codes` (Appendix B),
- `distance_to_pass` (how close it was),
- `recommended_protocol` (what to do next),
- `reopen_triggers` (what should trigger re-evaluation).

Important: an “error” is treated as a **negative hypothesis** and may become valid with new data.

---

## 7) Agent Feedback Loop (personalization layer)

Appendix E defines:
- the **AgentFeedbackEnvelope** format,
- required fields the agent must receive,
- override events and “human challenge” rules.

Core idea: SVP is the **filter**, the personal agent is the **iteration engine**.

---

## 8) Policy, integrity, reproducibility

### 8.1 PolicyPack (single YAML)
`svp/policy/SVP-EVAL_PolicyPack_v0.1.2_updated_fixed_step6.yaml` defines:
- required keys (MUST exist in any implementation),
- thresholds,
- mandatory side outputs,
- fail-closed behavior.

### 8.2 MANIFESTs and hash verification
- Bundle manifest: `svp/audit/MANIFEST.hashes.bundle.v0.2.11.json`
- Repo manifest: `svp/audit/MANIFEST.hashes.repo.v0.2.11.json`

**Linux/macOS:**
```bash
sha256sum <file>
```

**Windows PowerShell:**
```powershell
Get-FileHash \<file\> \-Algorithm SHA256
```

---

## 9) Updating via delta bundle

If you already have a working checkout and only want changes:

1. unpack the `SVP_update_bundle_...zip` over your working directory  
2. read `PATCHLIST` (what changed)  
3. verify `MANIFEST` (hashes must match)  
4. (optionally) run your local implementation checks bound to PolicyPack

---

## 10) About “diversity loss” risk

Stricter optimizations (near-dup in Step 4, evidence structuring in Step 5, coupling graph pinning, fail-closed in Step 6\) can reduce surface variety of candidates.

Compensators:

* SideRecords \+ banks,  
* agent protocols (especially CREATIVE/LEARNING),  
* reopen triggers \+ human challenge.

---

## 11) Quick index of primary files

* Canon: `svp/core/SVP_Canon_Index_v0.2.11.md`  
* PolicyPack: `svp/policy/SVP-EVAL_PolicyPack_v0.1.2_updated_fixed_step6.yaml`  
* Side outputs: `svp/appendices/Appendix_D_SideOutputs_Banks_Contract_v0.1.md`  
* Agent loop: `svp/appendices/Appendix_E_Agent_Feedback_Loop_Contract_v0.1.md`  
* Steps: `svp/steps/STEP_1_...` … `STEP_6_...`  
* Audit: `svp/audit/STYKOVKA_CrossCheck_Report_v0.2.11.md`, `svp/audit/PATCHLIST_v0.2.11.md`

---

## 12) Status and license

* **Status:** engineering specification \+ canonical version/hash registry  
* **License:** TBD (add on publication)

