# MAMAI — Quality Evaluation

*System under evaluation: Gemma 4 E4B + RAG over WHO / Tanzania MOH / Zanzibar MOH guidelines, deployed as a medical-advice chatbot for nurses and midwives in Zanzibar*

*Domain scope: OBGYN, neonatal / infant, reproductive health*

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

**Primary metric: sentence-level support rate.** For each answer, check what fraction of sentences are entailed by the retrieved context using MiniCheck. Two pipeline variants are used, differing in whether a decomposition step precedes verification.

MiniCheck is a 7B NLI model trained on synthetic data that reflects LLM hallucination patterns. It takes a (claim, context) pair and outputs a continuous probability score (0–1); the default threshold is 0.5. It runs fully offline, requires no API, and is deterministic. MiniCheck is preferred over LLM-as-judge as the primary signal for three reasons: (1) **scale** — the full eval set spans thousands of answers across all model × RAG conditions, making per-answer API costs prohibitive; (2) **reproducibility** — MiniCheck produces the same score for the same input every time; (3) **speed** — local inference enables fast iteration when re-evaluating after retriever or prompt changes.

**Known limitation — partially correct sentences.** For a sentence that encodes multiple claims where some are supported and some are not, MiniCheck returns one blended probability for the whole sentence. A partially wrong sentence can score above 0.5 and be counted as supported. This is particularly relevant for dense clinical sentences that pack multiple dosing or procedural claims into one (e.g. loading dose + maintenance dose). The sentence-level support rate metric will therefore be optimistic in the presence of such sentences. Report this as a known limitation alongside the scores; the calibration step (below) against a frontier LLM judge — which can label claims as *partially supported* — will surface how often this occurs.

**Pipeline 1 — MiniCheck directly (no decomposition).**
MiniCheck was designed to check sentence-level or paragraph-level text directly without requiring atomic decomposition. Split the answer into sentences using scispaCy (a biomedical-tuned tokenizer that correctly handles clinical abbreviations and terminology), then feed each sentence to MiniCheck against the retrieved context. Aggregate to a per-answer support rate. This pipeline is fully deterministic, requires no LLM calls, and is the fastest option.

**Pipeline 2 — Decompose first, then MiniCheck.**
For answers where a single sentence encodes multiple independent claims (common in clinical guidelines), sentence-level splitting may under-penalise hallucination embedded in otherwise faithful sentences. In this pipeline: (1) split into sentences with scispaCy, (2) decompose each sentence into atomic claims using MedScore — a Flan-T5 model fine-tuned specifically for medical claim decomposition, handling condition-dependent statements ("for diabetic patients…") and hedged recommendations that general decomposers miss, (3) run MiniCheck on each atomic claim. This yields finer-grained support rates at the cost of an additional small-model inference step. MedScore is local and deterministic; no LLM is required.

*Note: research ("Decomposition Dilemmas", 2024) shows that decomposition can hurt rather than help on conditional and hedged medical text, because fragments become unverifiable in isolation. Pipeline 1 should be the default; Pipeline 2 used selectively on longer, denser answers.*

**Calibration — LLM-as-judge (frontier model, batch).**
Run a frontier model (Claude / GPT-4) on a random subset: given the retrieved passage(s) and the generated answer, label each claim as *fully supported*, *partially supported*, or *unsupported*. Use 5–10 few-shot examples from the OBGYN / neonatal domain. If both pipeline outputs agree with the LLM judge on the validation subset (Spearman ρ > 0.7), MiniCheck scores are trusted at scale.

**Reporting.** Report sentence-level support rate (mean ± std) for both pipelines across the mamaretrieval query set under oracle context. Large gaps between Pipeline 1 and Pipeline 2 on the same answers indicate multi-claim sentences and inform which pipeline to use at scale.

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
