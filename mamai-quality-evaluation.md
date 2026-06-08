# MAMAI — Quality Evaluation

*System under evaluation: Gemma 4 E4B + RAG over WHO / Tanzania MOH / Zanzibar MOH guidelines, deployed as a medical-advice chatbot for nurses and midwives in Zanzibar*

*Domain scope: OBGYN, neonatal / infant, reproductive health*

> **Status (2026-06-08):** retriever evaluation (§2) and generator faithfulness (§3.1) are built and run. End-to-end MCQ (§4) is run. End-to-end open-ended (§4) is blocked on a closed-source judge decision. Live results live in `mamai-eval` (`configs/config-v0.2.0/reports/`) and on the `feat/faithfulness-eval` branch of the same repo (`docs/faithfulness-eval-v0.2.0.md`). Forward-looking plan: [`next-steps-2026-06.md`](next-steps-2026-06.md).

---

## 1. What is being evaluated

The deployed system is Gemma 4 E4B with RAG over a guideline corpus (WHO, Tanzania MOH, Zanzibar MOH, ACOG / RCOG / NICE / FIGO, EmONC handbook, Labour Care Guide, PCPNC, Helping Babies Breathe, national formularies).

### Evaluation methodology

This evaluation follows a bottom-up component methodology, validating each part of the RAG pipeline independently before assessing the integrated system:

1. **Retriever** — evaluated on mamaretrieval. Before trusting the end-to-end system, we verify that the retriever reliably surfaces relevant guideline content with high precision and recall in the top-k results. A weak retriever makes model evaluation meaningless.
2. **Generator** — evaluated in isolation under oracle context. Given that the retriever works, we verify that Gemma uses the retrieved context faithfully and stably, without hallucinating beyond it.
3. **End-to-end system** — evaluated on mamabench. With both components validated, we measure the complete RAG pipeline against external benchmarks and compare against larger models with and without retrieval.

Two benchmark artifacts support this:

- **mamaretrieval** — a retrieval benchmark of `(query, relevant_chunk_ids)` pairs built from the guideline corpus itself. Used for retriever evaluation. See `mamaretrieval.md`.
- **mamabench** — a filtered QA benchmark (MCQ + open-ended + safety) built from existing expert-validated sources. Used for end-to-end scoring. See `mamabench.md`.

---

## 2. Retriever evaluation (on mamaretrieval)

### 2.1 Retriever-only metrics

Run the deployed retriever (same embedding model, same chunker, same top-k) over the mamaretrieval query set. Score against `relevant_chunk_ids`:

- **Recall@k, Precision@k** at k = 1, 3, 5, 10.
- **MRR** (Mean Reciprocal Rank).
- **nDCG@k**.
- **Hit rate** — fraction of queries where any relevant chunk is retrieved.

Report separately on the LLM-generated query set and the hand-written query set. Large gaps between the two are a signal that LLM-generated queries are unrealistic.

### 2.2 Retriever comparators

If retrieval performance is a concern, compare the deployed retriever against:

- BM25 baseline
- A different embedding model (e.g., E5, BGE-M3, medical-domain embedding)
- Hybrid (BM25 + dense)
- Re-ranker added on top
- Other advanved retreiver models (like https://huggingface.co/lightonai/LateOn)

Same query set, same corpus, same k. Report the same metrics.

---

## 3. Generator evaluation — LLM (Gemma 4) behaviour

This section probes how Gemma 4 behaves once retrieval is held fixed. All evaluations use oracle context from mamaretrieval — retrieval quality is the concern of §2, not §3. The three questions are: does the model use the context faithfully, does it do so consistently, and does it handle deployment-specific (edge) cases correctly?

We could also use other models or better models to compare with Gemma 4 on model behaviour.

### 3.1 Faithfulness / groundedness

Given the oracle retrieved context, does Gemma stick to it or hallucinate around it? This is generator behaviour, not retrieval behaviour — retrieval quality and generation faithfulness correlate weakly, so this must be measured independently.

#### Judge-model selection

Three candidates were considered; two were rejected with reasons worth keeping:

1. **MiniCheck (Bespoke-MiniCheck-7B)** — built and smoke-tested. Rejected: at 7B it is small for the task, and being a pure NLI classifier it produces no reasoning trace, so verdicts can't be audited row-by-row.
2. **Qwen3.5-397B-A17B** — rejected for **circularity**. This is the same model that produced the mamaretrieval relevance labels that *built* the oracle. Using it again to judge faithfulness against that oracle means a shared blind spot would inflate the score undetectably.
3. **Patronus Lynx 70B** (`PatronusAI/Llama-3-Patronus-Lynx-70B-Instruct`) — **chosen.** Open weights, purpose-built for RAG hallucination detection, medical-domain trained (PubMedQA), emits bullet-point reasoning, Llama-3 family (independent of the Qwen oracle judge). Limitation: holistic per-response PASS/FAIL, not per-claim.

#### Pipeline

Three stages, all on the EPFL cluster:

| Stage | Tool | Output | Compute |
|---|---|---|---|
| 1. Oracle build | `build_oracle.py` (HF loader over mamaretrieval) | top-3 oracle chunks per query, capped at deployment `top_k = 3` | local |
| 2. Generation | `eval_faithfulness.py` (llama-cpp, Gemma 4 E4B Q4_0, deployment params T=1.0, top_p=0.95, top_k=64) | per-query Gemma response under oracle context | 1×A100, ~1.3 s/query |
| 3. Faithfulness scoring | `score_lynx.py` (Lynx 70B via vLLM) | per-response PASS/FAIL + reasoning | 2×A100, ~27 min for ~2,500 responses |

Oracle = mamaretrieval chunks with relevance `score ≥ 5` (the threshold the relevance judge was validated at against Claude Opus). Queries with no `score ≥ 5` chunk are excluded; queries with >3 are capped to the top 3 to match deployed retrieval depth.

#### Calibration with a frontier judge (required, not optional)

Lynx is a holistic detector and tends to over-flag — raw FAIL counts are **not** the true-hallucination rate. The calibration step takes the FAIL pool and asks a frontier LLM to:

1. **Categorise** each FAIL into `{contradiction, unsupported_addition, omission, unclear, parse-fail}`. Only the first two are genuine faithfulness failures — `omission` (incomplete-but-correct answer) and `parse-fail` (scoring artefact) are subtracted from the raw FAIL count.
2. **Independently verdict** a sample of FAILs as PASS/FAIL/unclear, to estimate Lynx's precision on its FAILs.

The calibrated true-hallucination rate is then `(genuine FAILs from categorisation) × (Lynx precision from independent verdict) / total responses`.

**Why a frontier judge is non-negotiable here.** Open-weight judges were tried as a cheaper substitute (gpt-oss-120b at `reasoning_effort=high`) and **failed both pre-committed gates** against the Claude Opus baseline:
- Categorisation: only the `unclear` bucket was within ±20% tolerance; gpt-oss reassigned 24/48 Claude `contradiction` labels to other buckets.
- Calibration agreement: 75/100 vs ≥90 required. 18 of the 25 disagreements are PASS / FAIL drift on cases the rubric explicitly says are PASS — refusals with escalation, partial-omission answers, clarification questions.
- The calibrated true-hallucination rate would read **11.20% vs Claude's 0.33%** — ~34× off, pure rubric-drift, not data signal.

Open weights rubber-stamp incompleteness and refusals as faithfulness failures despite the rubric saying PASS. The final calibration requires a closed-source frontier judge (currently planned: GPT-5.5).

#### Results

Two pipeline passes complete:

| Oracle | Queries | Raw Lynx PASS rate | 95% CI |
|---|---:|---:|---|
| v0.1.0 (top-3 union, score ≥ 5) | 2,659 | **94.55%** (2,514 / 145) | [93.6, 95.4] |
| v0.2.0 (top-20 union, score ≥ 5) | 2,989 | **94.18%** (2,815 / 174) | [93.3, 95.0] |

CIs overlap heavily — the richer v0.2.0 oracle did not shift the raw headline.

**v0.1.0 calibration (Claude Opus baseline, n=100 sample + full FAIL categorisation):**

- Lynx precision on its FAILs: ~6% (heavy over-flagger).
- Lynx recall on PASS rows: 0/50 misses (no false negatives in the sample).
- FAIL categorisation of all 145 raw FAILs: 48 contradictions + 22 unsupported additions + 35 omissions + 40 unclear/parse-fail. Only the first two count as faithfulness failures under the rubric.
- **Calibrated true-hallucination rate ≈ 0.3%.**

**v0.2.0 calibration: pending.** Blocked on the same closed-source judge decision that blocks the open-ended rubric track in §4. See [`next-steps-2026-06.md`](next-steps-2026-06.md) §"Cross-cutting decisions" for the consolidated budget ask.

#### Side-finding: 6 self-contradictory oracle contexts

The categorisation pass surfaced 6 queries whose oracle chunks contradict each other — i.e. the benchmark itself contains internally inconsistent ground truth (e.g. different misoprostol doses across two sources, both labelled relevant). Filed upstream as [`mamaretrieval` issue #18](https://github.com/nmrenyi/mamaretrieval/issues/18) for cleanup.

#### What this leaves open

Oracle-context faithfulness is essentially solved. The unanswered question is **faithfulness under real (non-oracle) retrieval**: when the input is the deployed gecko top-3 rather than curated `score ≥ 5` chunks, does the faithfulness rate collapse? This is the experiment that can fire the decision gate in [`mamai-finetuning-plan.md`](mamai-finetuning-plan.md). Until it runs, the fine-tuning plan stays shelved.

### 3.2 Stability

MCQ is retained in §4 as a reference metric for benchmarking, but faithfulness and stability are only measured on open-ended answers — the format that reflects real deployment, where nurses ask free-form clinical questions.

Stability is measured entirely through the MiniCheck claim-support rate from §3.1: a stable model should return consistent faithfulness scores under perturbation, regardless of sampling noise or surface rephrasing. Three probes:

- **Prompt sensitivity:** rephrase each question (MedFuzz-style paraphrases or hand-written) and retrieve the same context. Report the delta in MiniCheck score between original and paraphrased query. A nurse rephrasing a question slightly is a real deployment event — if faithfulness drops under paraphrasing, the model is relying on surface wording rather than the context.
  - first rewrite the query? Would it be better?
- **Run-to-run variance:** run Gemma N times on the same open-ended question with the same retrieved context (temperature > 0). Report the std of MiniCheck claim-support rate across runs. A stable model should produce consistently faithful answers regardless of sampling.
- **Greedy vs sampled:** compare MiniCheck score at temperature-0 vs temperature-0.7 on a subset. Informs deployment temperature choice — if sampling introduces meaningful faithfulness loss, greedy decoding is preferable.

### 3.3 Deployment integrity checks

These catch RAG-specific failures that generic metrics miss.

- **Functionality-level accuracy.** The core function of this RAG system is to provide guideline-backed answers. This checks whether that function executes correctly: when the model cites a guideline, does the cited source exist in the corpus, and does it actually say what the model claims? A failure here means the system is not delivering its primary value proposition — it is fabricating the very evidence it is supposed to be grounded in.
- **Contradiction among guidelines.** A curated set (~20–30 cases) of clinically important points where guidelines diverge — e.g. WHO global recommendations vs Zanzibar / Tanzania MOH protocols on misoprostol dosing, magnesium sulfate regimens, PMTCT, anaemia thresholds, malaria in pregnancy. These cases are identified by reading the guidelines directly and require clinical expertise to compile. Two things are evaluated on this set: (1) does Gemma acknowledge the conflict or silently pick one source? (2) when Gemma picks the local guideline over the global standard, does it score correctly against a local-aligned answer key rather than the benchmark key? The gap between benchmark-aligned and local-aligned scores is a deployment-relevant signal, not a failure.

---

## 4. End-to-end evaluation — mamabench

### 4.1 Evaluation matrix

Full matrix, reported per cell:

| Model | No RAG | + RAG (guideline corpus) |
|---|---|---|
| **Gemma 4 E4B** (deployment target) | baseline capability | **your deployed system** |
| Medical fine-tuned (MedGemma 4B, Meditron 70B) | fine-tuning vs. RAG — same compute, different adaptation strategy | fine-tuning + RAG ceiling |
| Mid-size open model (Llama 3 70B / Qwen 2.5 72B) | open-weights comparator | bigger-model-with-same-knowledge |
| Frontier model (GPT-5, Claude, Gemini 2.5) | raw capability ceiling | overall ceiling |

Six comparisons extracted from the matrix:

1. **Gemma-no-RAG → Gemma-RAG:** value added by your retrieval. Core RAG ablation.
2. **Gemma-RAG → MedGemma-no-RAG:** the fine-tuning vs. RAG headline — at the same 4B edge-device scale, does medical fine-tuning without retrieval match a general model with guideline RAG?
3. **Gemma-RAG → Frontier-no-RAG:** the deployability headline — can a 4B edge model with guideline retrieval match a frontier model working from pretraining alone?
4. **Frontier-no-RAG → Frontier-RAG:** does the corpus add information beyond what frontier models already know from pretraining?
5. **Gemma-RAG → Mid-size open model / Frontier-RAG:** model-size headroom when both have the same knowledge.
6. **Gemma-no-RAG → Mid-size open model / Frontier-no-RAG:** raw capability gap at the task.

**Methodological constraint.** Keep the RAG pipeline identical across all "+ RAG" rows: same retriever, same embedding model, same chunking, same top-k, same prompt template. Only the generator changes.

**Metrics.**

- MCQ: accuracy, calibration (Brier score, ECE).
- Open-ended: HealthBench-style rubric scores (factuality, reasoning, harm, omission, guideline adherence).
- Safety: pass rate on EquityMedQA / FairMedQA items, robustness under MedEqualQA / MedFuzz perturbations.

### 4.2 Corpus-composition ablation

Does corpus breadth matter, or is any single guideline source enough? Run Gemma + RAG over:

- WHO-only corpus
- Tanzania MOH-only corpus
- Zanzibar MOH-only corpus
- Full corpus (production)

Compare to Gemma-no-RAG and Gemma-full-RAG. Identifies which sources drive utility and informs corpus-maintenance priorities.

---

## 5. Reporting notes

If necessary, break results down along **category** (OBGYN vs neonatal vs reproductive; MCQ vs open-ended vs safety).

Include a **contamination caveat** for MedMCQA / MedQA / MMLU (likely in Gemma's pretraining). Run MedFuzz paraphrases on a subset to show contamination-adjusted numbers alongside raw numbers.

Report confidence intervals on all subsampled conditions (frontier-model rows, stability probes, deployment integrity checks).

---

## 6. Narrative target

Although we haven't run the results yet and the conclusion should be strictly following the results, the final outcome of this comprehensive evaluation goes like this:

> "Retrieval metrics on mamaretrieval show the pipeline hits Recall@5 of R and MRR of M over the guideline corpus. Compared to baseline retrievers (BM25, etc.), it performs better/worse.

> Under oracle retrieval, Gemma achieves a claim-level support rate of F, indicating the generator faithfully uses the retrieved context. The model is stable under paraphrasing and re-sampling (variance < V).

> On mamabench, Gemma 4 E4B without retrieval scores X%. With RAG over the WHO / Tanzania MOH guideline corpus, it reaches Y%. MedGemma 4B without retrieval scores Z_med%, directly testing whether medical fine-tuning at the same edge-device scale can substitute for RAG — Gemma + RAG outperforms / underperforms MedGemma alone. Meditron 70B without retrieval scores Z_mit%, the medical fine-tuned mid-size comparator. GPT-5 without retrieval scores Z%, GPT-5 + RAG reaches W%, the overall ceiling. On the local-guideline-aligned safety subset, the RAG system outperforms / underperforms all no-RAG baselines, including frontier and medical fine-tuned models."
