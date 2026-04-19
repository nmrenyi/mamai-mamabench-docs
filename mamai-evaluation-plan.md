# MAMAI — Evaluation Plan

*System under evaluation: Gemma 4 E4B + RAG over WHO / Tanzania MOH / Zanzibar MOH guidelines, deployed as a medical-advice chatbot for nurses and midwives in Zanzibar*

*Domain scope: OBGYN, neonatal / infant, reproductive health*

---

## 1. What is being evaluated

The deployed system is Gemma 4 E4B with RAG over a guideline corpus (WHO, Tanzania MOH, Zanzibar MOH, ACOG / RCOG / NICE / FIGO, EmONC handbook, Labour Care Guide, PCPNC, Helping Babies Breathe, national formularies). The evaluation answers three questions:

1. How well does the end-to-end system perform on OBGYN / neonatal / reproductive-health QA, and how does it compare to larger models with and without the same retrieval?
2. How well does the retriever alone surface relevant guideline content?
3. How does the Gemma 4 generator behave — under varying retrieval quality, under paraphrasing and re-sampling, and on deployment-specific integrity checks?

Two benchmark artifacts support this:

- **mamabench** — a filtered QA benchmark (MCQ + open-ended + safety) built from existing expert-validated sources. Used for end-to-end scoring. See `mamabench.md`.
- **mamaretrieval** — a retrieval benchmark of `(query, relevant_chunk_ids)` pairs built from the guideline corpus itself. Used for retriever scoring and generator-behaviour diagnostics. See `mamaretrieval.md`.


## 2. Evaluation on end-to-end QA -- mamabench

### 2.1 Evaluation matrix

Full matrix, reported per cell:

| Model | No RAG | + RAG (guideline corpus) |
|---|---|---|
| **Gemma 4 E4B** (deployment target) | baseline capability | **your deployed system** |
| Mid-size open model (Llama 3 70B / Qwen 2.5 72B) | open-weights comparator | bigger-model-with-same-knowledge |
| Frontier model (GPT-5, Claude, Gemini 2.5) | raw capability ceiling | overall ceiling |

Five comparisons extracted from the matrix:

1. **Gemma-no-RAG → Gemma-RAG:** value added by your retrieval. Core RAG ablation.
2. **Gemma-RAG → Frontier-no-RAG:** the deployability headline — can a 4B edge model with guideline retrieval match a frontier model working from pretraining alone?
3. **Frontier-no-RAG → Frontier-RAG:** does the corpus add information beyond what frontier models already know from pretraining?
4. **Gemma-RAG → Mid-size open model / Frontier-RAG:** model-size headroom when both have the same knowledge.
5. **Gemma-no-RAG → Mid-size open model / Frontier-no-RAG:** raw capability gap at the task.

**Methodological constraint.** Keep the RAG pipeline identical across all "+ RAG" rows: same retriever, same embedding model, same chunking, same top-k, same prompt template. Only the generator changes.

**Metrics.**

- MCQ: accuracy, calibration (Brier score, ECE).
- Open-ended: HealthBench-style rubric scores (factuality, reasoning, harm, omission, guideline adherence).
- Safety: pass rate on EquityMedQA / FairMedQA items, robustness under MedEqualQA / MedFuzz perturbations.

### 2.2 Corpus-composition ablation

Does corpus breadth matter, or is any single guideline source enough? Run Gemma + RAG over:

- WHO-only corpus
- Tanzania MOH-only corpus
- Zanzibar MOH-only corpus
- Full corpus (production)

Compare to Gemma-no-RAG and Gemma-full-RAG. Identifies which sources drive utility and informs corpus-maintenance priorities.

---

## 3. Evaluation on retrieval module

### 3.1 Retriever-only metrics

Run the deployed retriever (same embedding model, same chunker, same top-k) over the query set. Score against `relevant_chunk_ids`:

- **Recall@k, Precision@k** at k = 1, 3, 5, 10.
- **MRR** (Mean Reciprocal Rank).
- **nDCG@k**.
- **Hit rate** — fraction of queries where any relevant chunk is retrieved.

Report separately on the LLM-generated query set and the hand-written query set. Large gaps between the two are a signal that LLM-generated queries are unrealistic.

### 3.2 Retriever comparators

If retrieval performance is a concern, compare the deployed retriever against:

- BM25 baseline
- A different embedding model (e.g., E5, BGE-M3, medical-domain embedding)
- Hybrid (BM25 + dense)
- Re-ranker added on top

Same query set, same corpus, same k. Report the same metrics.

---

## 4. Evaluation on Gemma 4 Behaviour

This section probes how Gemma 4 behaves once retrieval is held fixed. Context ablations, stability, faithfulness, and deployment integrity checks all answer "how does the generator use the context it's given?"

We could also use other models or better models, to compare with Gemma 4 on the model behaviour.

### 4.1 Faithfulness / groundedness

Given the oracle retrieved context, does Gemma stick to it or hallucinate around it? This is generator behaviour, not retrieval behaviour — retrieval quality and generation faithfulness correlate weakly, so this must be measured independently.

**Primary metric: claim-level support rate.** Decompose each answer into atomic claims, then check what fraction of claims are entailed by the retrieved context. Two complementary automatic methods are used.

**Method A — MiniCheck (NLI-based, offline). Primary method.**
MiniCheck is a 7B NLI model trained on synthetic data that reflects LLM hallucination patterns. For each answer–context pair: (1) decompose the answer into atomic claims, (2) run MiniCheck to classify each claim as entailed or not entailed by the context, (3) compute the fraction of claims entailed. MiniCheck runs fully offline, requires no API, and can be applied to every answer in the evaluation set.

MiniCheck is preferred as the primary signal over LLM-as-judge for three reasons: (1) **scale** — the full eval set spans thousands of answers across all model × RAG conditions, making per-answer API costs prohibitive; (2) **reproducibility** — MiniCheck is deterministic, producing the same score for the same input, whereas LLM-as-judge is stochastic and would require multiple runs per answer to stabilise; (3) **speed** — local inference enables fast iteration when re-evaluating after retriever or prompt changes. LLM-as-judge is strictly more capable at nuanced clinical reasoning, but MiniCheck is the right choice for a scalable, reproducible, paper-ready pipeline.

**Method B — LLM-as-judge (frontier model, batch). Calibration method.**
Run a frontier model (Claude / GPT-4) on a random subset with the following prompt structure: given the retrieved passage(s) and the generated answer, list every claim in the answer and label each as *fully supported*, *partially supported*, or *unsupported* by the passages. Use 5–10 few-shot examples drawn from the OBGYN / neonatal domain to anchor the judge to clinical terminology. Aggregate to a per-answer support rate. This method is used to calibrate MiniCheck: if the two methods agree on a validation subset (Spearman ρ > 0.7), MiniCheck scores are trusted at scale.

**Reporting.** Report claim-level support rate (mean ± std) across the mamaretrieval query set under oracle context. All generator behaviour evaluations (§4.1, §4.2, §4.3) use oracle context — retrieval quality is evaluated separately in §3.

### 4.2 Stability

MCQ is retained in §2.1 as a reference metric for benchmarking, but faithfulness and stability are only measured on open-ended answers — the format that reflects real deployment, where nurses ask free-form clinical questions.

Stability is measured entirely through the MiniCheck claim-support rate from §4.1: a stable model should return consistent faithfulness scores under perturbation, regardless of sampling noise or surface rephrasing. Three probes:

- **Run-to-run variance:** run Gemma N times on the same open-ended question with the same retrieved context (temperature > 0). Report the std of MiniCheck claim-support rate across runs. A stable model should produce consistently faithful answers regardless of sampling.
- **Prompt sensitivity:** rephrase each question (MedFuzz-style paraphrases or hand-written) and retrieve the same context. Report the delta in MiniCheck score between original and paraphrased query. A nurse rephrasing a question slightly is a real deployment event — if faithfulness drops under paraphrasing, the model is relying on surface wording rather than the context.
- **Greedy vs sampled:** compare MiniCheck score at temperature-0 vs temperature-0.7 on a subset. Informs deployment temperature choice — if sampling introduces meaningful faithfulness loss, greedy decoding is preferable.

### 4.3 Deployment integrity checks

Hand-reviewed subset. These catch RAG-specific failures that generic metrics miss.

- **Functionality-level accuracy.** The core function of this RAG system is to provide guideline-backed answers. This checks whether that function executes correctly: when the model cites a guideline, does the cited source exist in the corpus, and does it actually say what the model claims? A failure here means the system is not delivering its primary value proposition — it is fabricating the very evidence it is supposed to be grounded in.
- **Contradiction among guidelines.** A curated set (~20–30 cases) of clinically important points where guidelines diverge — e.g. WHO global recommendations vs Zanzibar / Tanzania MOH protocols on misoprostol dosing, magnesium sulfate regimens, PMTCT, anaemia thresholds, malaria in pregnancy. These cases are identified by reading the guidelines directly and require clinical expertise to compile. Two things are evaluated on this set: (1) does Gemma acknowledge the conflict or silently pick one source? (2) when Gemma picks the local guideline over the global standard, does it score correctly against a local-aligned answer key rather than the benchmark key? The gap between benchmark-aligned and local-aligned scores is a deployment-relevant signal, not a failure.

---

## 5. Reporting notes

If necessary, break results down along **category** (OBGYN vs neonatal vs reproductive; MCQ vs open-ended vs safety).

Include a **contamination caveat** for MedMCQA / MedQA / MMLU (likely in Gemma's pretraining). Run MedFuzz paraphrases on a subset to show contamination-adjusted numbers alongside raw numbers.

Report confidence intervals on all subsampled conditions (frontier-model rows, context ablations, stability probes, deployment integrity checks).

---

## 6. Narrative target

Although we haven't run the results yet and the conclusion should be strictly following the results, the final outcome of this comprehensive evaluation goes like this:

> "Gemma 4 E4B without retrieval scores X% on mamabench. With RAG over the WHO / Tanzania MOH guideline corpus, it reaches Y%, closing most of the gap to GPT-5 without retrieval (Z%) and LlaMa / Medgemma / Meditron (Z'%). 

> GPT-5 + our RAG reaches W%, LlaMa + out RAG reaches W'%, indicating further headroom if compute were available. 

> Retrieval metrics on mamaretrieval show the pipeline hits Recall@5 of R and MRR of M over the guideline corpus. Compared to baseline retrievers (BM25, etc.), it performs better/worse.

> Under oracle retrieval, Gemma reaches O%, indicating the retriever — not the generator — is the primary bottleneck / is already close to saturating the generator. The model is stable under paraphrasing and re-sampling (variance < V), and on the local-guideline-aligned safety subset outperforms / underperforms all no-RAG baselines, including frontier models."

