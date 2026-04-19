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

### 4.1 Faithfulness / groundedness

Given the oracle retrieved context, does Gemma stick to it or hallucinate around it?
It is generator behaviour, not retrieval behaviour.

### 4.2 Stability

Same input, run multiple times or lightly perturbed. Probes whether the system is reliable under normal deployment variance.

- **Run-to-run variance:** sample the model N times (temperature > 0 or different seeds) on a 100–200 item subset. For MCQ, report the fraction of items where Gemma picks a different option across runs. For open-ended, report rubric-score variance across runs.
- **Prompt sensitivity:** rephrase each question (hand-written or MedFuzz-style paraphrases). Report consistency of the answer across paraphrases. A nurse rephrasing a question slightly is a real deployment event.
- **Answer-order sensitivity (MCQ only):** shuffle A/B/C/D options. Report the fraction of items where the selected answer flips under reorder. A stable model should not flip.
- **Greedy vs sampled:** compare temperature-0 vs temperature-0.7 behaviour on a subset. Informs deployment temperature choice.

### 4.3 Deployment integrity checks

Hand-reviewed subset. These catch RAG-specific failures that generic metrics miss.

- **Guideline attribution accuracy.** When the model cites a guideline, is the citation real and correct?
- **Contradiction handling.** When retrieved chunks conflict (e.g., WHO vs ACOG on a threshold), does the model acknowledge the conflict or silently pick one?
- **Local-guideline-aligned score.** For items where a public-benchmark "correct" answer conflicts with WHO / Tanzania MOH (e.g., misoprostol PPH dosing, magnesium sulfate regimens, PMTCT protocols, anaemia thresholds, malaria in pregnancy), report a second score aligned to local guidelines rather than the benchmark key. This disagreement is a deployment-relevant feature of RAG on local guidelines, not a bug.

### 4.4 Anti-noise ability

Run Gemma 4 on mamaretrieval queries with the context source swapped out:

| Condition | Context fed to Gemma | What it diagnoses |
|---|---|---|
| **Oracle context** | the gold chunk(s) directly | ceiling performance if retrieval were perfect — isolates generator capability from retriever failures |
| **No context** | empty | baseline without retrieval |
| **Deployed retrieval** | top-k from the production retriever | realistic operating condition |
| **Random context** | k random chunks from the corpus | robustness to irrelevant retrieval — score should stay near no-context, not collapse |
| **Degraded context** | lower-ranked chunks, or chunks from a deliberately poor retriever | graceful-degradation check |

Metrics: 4.1, 4.2, 4.3

These ablations are well-defined on mamaretrieval (gold chunks exist by construction) and ill-defined on mamabench (most items have no identifiable gold chunk in the guideline corpus).


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

