# MAMAI — RAG-Aware Fine-Tuning Plan

*System under fine-tuning: Gemma 3n E4B, the generator of the deployed RAG medical-advice chatbot for nurses and midwives in Zanzibar*

*Domain scope: OBGYN, neonatal / infant, reproductive health*

> **Status (2026-06-08): GATED — not currently triggered.** Oracle-context faithfulness on Gemma 4 E4B is **~0.3% calibrated true-hallucination rate** (raw Lynx 94.55% PASS at v0.1.0; see [`mamai-quality-evaluation.md` §3.1](mamai-quality-evaluation.md#31-faithfulness--groundedness)). The dominant failure mode this plan hedges against — Pandey "irrelevant generation" — is empirically absent under oracle context. **Do not execute** until the real-retrieval faithfulness probe (faithfulness vs deployed gecko top-3, see [`next-steps-2026-06.md`](next-steps-2026-06.md)) shows a material collapse vs the oracle headline. The plan below is preserved as a design reference for that contingency.

---

## 1. Motivation and decision gate

The literature (`rag-small-vs-large-literature.md`) raises one structural risk for MAMAI: a ~4B model running standard RAG without RAG-aware fine-tuning may fail to use retrieved context — Pandey et al. (arXiv:2603.11513, 2026) measured a net-negative trade-off for sub-7B models on open-domain QA. The "Diminishing Returns" study (arXiv:2502.11400, 2025) reinforces that RAG-aware fine-tuning pays off **most** exactly at small scale: adversarial/robust-RAG training lifts a 7B model far more than an 8B or larger model. A 4B generator is squarely in the range where this training matters.

However, MAMAI's measured faithfulness under oracle context is ~97% (MiniCheck sentence-level support rate). That is close to a direct refutation of Pandey's *dominant* failure mode — "irrelevant generation," where the model ignores retrieved passages. So the core context-utilization problem appears largely solved already, and this plan should **not** be executed speculatively.

This plan therefore targets the behaviors that 97%-faithfulness-under-oracle does **not** cover, and is gated on evidence.

### Decision gate

Run the fine-tuning pipeline only if one or more of these holds, measured by the existing evaluation plan:

| Signal | Source | Triggers fine-tuning if |
|---|---|---|
| RAG accuracy lift | §4.1 Gemma-no-RAG → Gemma-RAG | RAG is flat or net-negative on mamabench |
| Faithfulness under *real* (non-oracle) retrieval | §3.1 with deployed top-k | Support rate drops materially vs. the ~97% oracle figure |
| Distractor sensitivity | new probe (see §6) | Faithfulness/accuracy collapses when noisy chunks are present |
| Unsafe non-abstention | §3.3, safety set | Model answers confidently when retrieval misses |
| Citation fabrication | §3.3 citation functionality check | Cited sources do not exist or do not support the claim |

If RAG already lifts accuracy, faithfulness holds under real retrieval, and abstention/citation pass — fine-tuning is a *refinement*, not a rescue, and can be scoped down to §8 Phase 1 only.

---

## 2. Design principle — one dataset, four behaviors

The four needs — context utilization, distractor robustness, failure fallback, citation discipline — are all answers to the same question: *how should the generator behave given a set of retrieved chunks?* They can therefore be trained by **a single supervised fine-tuning (SFT) dataset** whose examples are a deliberate mixture of archetypes, rather than four separate training runs.

**Backbone: RAFT** (Zhang et al., arXiv:2403.10131, 2024). RAFT is chosen as the spine because it is the only verified method in this space that is simultaneously: (a) validated on medical QA (PubMedQA); (b) plain-text — **no tokenizer or vocabulary changes**, unlike Self-RAG / Open-RAG, which is essential for an edge-deployed model; (c) QLoRA-compatible; (d) trains context use, distractor handling, and verbatim-evidence grounding in one recipe.

RAFT alone is insufficient on three axes, so it is augmented:

| Behavior | Backbone | Augmented with | Why the augmentation |
|---|---|---|---|
| Context utilization | RAFT grounded examples | — | RAFT's core recipe |
| Distractor robustness | RAFT distractors (relevant-but-no-answer) | **RAAT** counterfactual distractors (arXiv:2405.20978) | RAAT shows counterfactual/misleading noise causes the largest damage (~−13%); RAFT's random distractors do not cover it |
| Failure fallback | RAFT distractor-only split | **abstain target** instead of forced answer; **R-Tuning** certain/uncertain split (arXiv:2311.09677); **DTA** four-quadrant framing (arXiv:2505.20871) | RAFT's distractor-only examples still expect an answer; a medical product must abstain |
| Citation discipline | RAFT `##begin_quote##` markers | **FRONT** verbatim-quote grounding (arXiv:2408.04568); **ALCE** metrics (arXiv:2305.14627) | RAFT markers are an internal scaffold, never measured; FRONT makes citations verifiable and ALCE makes them scorable |

This keeps the build to one QLoRA SFT run with no RL and no architecture change — appropriate for a small team. RL-based citation methods (Fine-grained Rewards, CANOE) and self-improvement loops (START, SelfCite) are noted as Phase 3 options only.

---

## 3. Training data construction

### 3.1 Source and questions

Built from the production guideline corpus (same chunks, same stable IDs as `mamaretrieval`). For each chunk, prompt a strong LLM for 3–5 nurse-voice questions spanning factual ("recommended misoprostol dose for PPH?"), procedural ("steps when…?"), and scenario ("a patient presents with X — what does the guideline recommend?") types. This reuses the `mamaretrieval` query-generation machinery.

### 3.2 Example archetypes and mixture

Each training example is `question + retrieved chunk set → target answer`. Four archetypes, mixed; proportions are starting points to tune on a held-out set (RAFT reports the optimal oracle fraction is dataset-dependent, 40–100%):

**A. Standard grounded (~55%)** — 1 oracle chunk + 2–4 distractors. Target: chain-of-thought answer that wraps the verbatim supporting span in `##begin_quote## … ##end_quote##` and cites the source chunk ID, FRONT-style. Trains context utilization + citation.

**B. Counterfactual-distractor (~20%)** — oracle present, but distractors include RAAT-style **entity-swapped** chunks (e.g. a near-duplicate of the oracle with the dose or threshold altered to a plausible wrong value). Target answer follows the *oracle* and ignores the swapped chunk. Trains the hardest, most clinically dangerous distractor type.

**C. Retrieval-failure → abstain (~20%)** — distractor-only, no oracle. Target: an explicit abstention ("The retrieved guidelines do not cover this — please consult a clinician / senior midwife"), not a fabricated answer. The certain/uncertain split for purely parametric questions follows R-Tuning's automatic labeling; DTA's four-quadrant scheme decides which examples get the abstain target.

**D. Guideline-conflict → multi-source attribution (~5%)** — the ~20–30 WHO vs. Tanzania/Zanzibar MOH divergence cases (misoprostol dosing, MgSO4 regimens, PMTCT, anaemia thresholds, malaria in pregnancy). One example per source; target answer **names both sources and their recommendations** rather than silently picking one. Trains citation discipline on the deployment-critical conflict cases.

### 3.3 Distractors and labels

Distractors for A/B/C are the deployed retriever's top-k for that question, excluding the oracle. Counterfactual distractors (B) are synthesized by entity/value swaps on the oracle and quality-checked. CoT targets and abstention phrasings are generated by a frontier model from `(question, oracle)` and reviewed; conflict cases (D) require clinical input to compile.

### 3.4 Scale and cost

~10k corpus chunks × 3 questions ≈ 30k examples. Generation with a small frontier model (e.g. GPT-4o-mini class) is ~$30–50. A Phase 1 minimal variant (Finetune-RAG, arXiv:2505.10792) showed +21pp from only ~1.6k examples — useful as a fast first cut.

---

## 4. Training configuration

| Setting | Value |
|---|---|
| Base model | Gemma 3n E4B (instruction-tuned) |
| Method | QLoRA SFT — no RL, no tokenizer change |
| LoRA config | r=16, α=32, target q/k/v/o/gate/up/down projections |
| Hardware | Single 24 GB GPU (RTX 4090 / A10G); ~2–4 h per 10k examples; Unsloth recommended (Gemma 3n E4B is QLoRA-tunable on a free Colab T4) |
| Distractor count | 2–4 per example (RAFT: 4 optimal for single-hop) |
| Oracle fraction | ~80% of A/B/D examples contain the oracle; tune on held-out data |

**Deployment quantization warning.** 4-bit (NF4) inference degrades long-context performance by up to ~59% (arXiv:2505.20276); 8-bit costs ~0.8%. Fine-tuning to improve context use is undermined if the deployed model is 4-bit. Deploy at 8-bit if device memory allows, and re-measure faithfulness post-quantization.

---

## 5. Evaluation of the fine-tuned model

Evaluate the fine-tuned generator against the unmodified Gemma 3n E4B on all four axes, reusing the existing plan where possible:

| Axis | Metric | Source |
|---|---|---|
| Context utilization | MiniCheck support rate; counterfactual substitution test (does output track a swapped value in context?) | `mamai-quality-evaluation.md` §3.1 |
| Distractor robustness | Faithfulness/accuracy with noisy top-k vs. oracle; accuracy on counterfactual-distractor probe | new probe |
| Failure fallback | Abstention precision (abstains when it should) **and** answer coverage (does not over-abstain) | safety set, §3.3 |
| Citation discipline | ALCE citation recall / precision / F1; citation functionality check (cited chunk exists and supports the claim) | §3.3 + ALCE harness |
| End-to-end | mamabench accuracy, no-RAG vs. RAG | §4.1 |
| Regression | General medical QA outside the guideline domain | contamination/MedFuzz subset |

Two non-negotiable checks: (1) **over-abstention** — every abstention method trades coverage for safety; report both precision and coverage and tune the distractor ratio against them. (2) **domain narrowing** — RAFT-tuned models can regress on out-of-domain medical QA; the regression row guards against shipping a model that is better on guidelines but worse generally.

---

## 6. Risks and limitations

- **No precedent.** No published paper fine-tunes Gemma 3n (or any Gemma) for text-RAG context utilization — this is genuinely new ground. All cited methods are validated at 7–13B; none below 7B. Citation discipline in particular is harder at 4B (weaker quote-copying fidelity); expect to lean on data quality over RL.
- **Speculative trigger.** The whole plan is gated on §1. If the decision-gate signals come back clean, executing this is wasted effort and risks regressions.
- **Counterfactual distractors** may not capture real retrieval noise; PrismRAG (arXiv:2507.18857) flags this and it is only validated at 70B.
- **Teacher dependence.** CoT/citation targets are distilled from a frontier model; quality is bounded by the teacher. START/SelfCite are teacher-free fallbacks if this is a concern.

---

## 7. Phased rollout

- **Phase 0 — Decision gate (§1).** Run the existing evaluation. Proceed only on a triggering signal.
- **Phase 1 — Minimal (Finetune-RAG style).** ~1.6–5k examples, archetypes A + C only, QLoRA. Fast proof that fine-tuning moves the targeted metrics. Cheap to abandon.
- **Phase 2 — Full RAFT+.** Full ~30k dataset, all four archetypes (A–D), the §5 evaluation. This is the core plan.
- **Phase 3 — Refinement (optional).** Only if Phase 2 citation/abstention metrics fall short: add rejection-sampling or fine-grained-reward citation training (arXiv:2402.04315), or DPO-based situated-faithfulness alignment (CR-DPO, arXiv:2410.14675). RL and reflection-token methods (Self-RAG, CANOE) remain out of scope for edge deployment unless Phase 3 fails.

---

## References

| Method | arXiv | Role in this plan |
|---|---|---|
| RAFT | 2403.10131 | Backbone SFT recipe |
| RAAT | 2405.20978 | Counterfactual distractor training |
| Chain-of-Note | 2311.09210 | "Insufficient info → reject" behavior |
| RetRobust | 2310.01558 | Low-data relevant/irrelevant mixing precedent |
| R-Tuning | 2311.09677 | Automatic certain/uncertain split for abstention |
| Divide-Then-Align | 2505.20871 | Four-quadrant abstention framing |
| Honest AI | 2410.09699 | Sub-10B "I don't know" + RAG precedent |
| FRONT | 2408.04568 | Verbatim fine-grained citation grounding |
| ALCE | 2305.14627 | Citation recall/precision/F1 metrics |
| Finetune-RAG | 2505.10792 | Phase 1 minimal recipe |
| Fine-grained Rewards | 2402.04315 | Phase 3 citation training option |
| CR-DPO (Situated Faithfulness) | 2410.14675 | Phase 3 alignment option |
| Diminishing Returns of Robust RAG Training | 2502.11400 | Evidence that robust FT matters most at ~4B |
| Pandey — small-model RAG utilization | 2603.11513 | The risk this plan hedges |
