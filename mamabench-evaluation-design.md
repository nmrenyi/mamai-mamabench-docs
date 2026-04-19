# mamabench — Evaluation Plan

*System under evaluation: Gemma 4 E4B + RAG over WHO / Tanzania MOH / Zanzibar MOH guidelines, deployed as a medical-advice chatbot for nurses and midwives in Zanzibar*

*Domain scope: OBGYN, neonatal / infant, reproductive health*

---

## 1. What is being evaluated

The deployed system is Gemma 4 E4B with RAG over a guideline corpus (WHO, Tanzania MOH, Zanzibar MOH, ACOG / RCOG / NICE / FIGO, EmONC handbook, Labour Care Guide, PCPNC, Helping Babies Breathe, national formularies). The evaluation answers three questions:

1. How well does the end-to-end system perform on OBGYN / neonatal / reproductive-health QA, and how does it compare to larger models with and without the same retrieval?
2. How well does the retriever alone surface relevant guideline content?
3. How does the Gemma 4 generator behave — under varying retrieval quality, under paraphrasing and re-sampling, and on deployment-specific integrity checks?

Two benchmark artifacts support this:

- **mamabench** — a filtered QA benchmark (MCQ + open-ended + safety) built from existing expert-validated sources. Used for end-to-end scoring.
- **mamaretrieval** — a retrieval benchmark of `(query, relevant_chunk_ids)` pairs built from the guideline corpus itself. Used for retriever scoring and generator-behaviour diagnostics.

### Corpus rule

Benchmark items are **never** placed in the RAG corpus. The RAG corpus contains only clinical knowledge sources. This applies to both mamabench and mamaretrieval.

### mamabench composition

- **MCQ set:** OBGYN + Pediatrics subsets of MedMCQA (native `subject_name`), filtered MedQA (keyword + tag), filtered MMLU-medical, PediatricsMQA (neonates, infants), PedMedQA (neonates 0–3 months, infants).
- **Open-ended set:** HealthBench items tagged with pregnancy (Z33.1), obstetric (O-chapter), or perinatal (P-chapter) ICD-10 codes, with HealthBench's physician-written rubrics.
- **Safety set:** EquityMedQA obstetric items, FairMedQA maternal items, plus MedEqualQA and MedFuzz perturbations applied to the MCQ anchor set.

### mamaretrieval composition

A shared corpus plus a query set with relevance labels. The corpus is the production RAG corpus chunked under a fixed scheme; each chunk has a stable ID.

**Query sources (two methods, complementary).**

- **LLM-generated (bulk):** for each chunk, prompt a strong LLM to generate 2–3 realistic nurse/midwife questions answered by that chunk. The chunk becomes a seed positive. Target: 1,000–3,000 queries.
- **Hand-written topic queries (small, trusted):** ~100 queries written by hand against the scope (PPH management, pre-eclampsia thresholds, neonatal resuscitation, danger signs, PMTCT, anaemia, malaria in pregnancy, etc.). Seed positives identified by reading the corpus.

**Relevance labeling — ensuring completeness.**

The core challenge in building a retrieval benchmark is that each query may be answered by multiple chunks, not just one, and missing positives bias retriever scores pessimistically and non-uniformly across retrievers. Complete labels cannot be guaranteed but can be approximated via TREC-style pooling plus systematic expansion.

Pipeline per query:

1. **Seed positive.** Method A: the source chunk the query was generated from. Method B: positives identified by the annotator while writing the query.
2. **Pool candidates across retrievers.** Run the query through 3–4 strong retrievers (BM25, at least one alternative embedding model, optionally hybrid). Union the top-10 results, dedupe. Yields ~20–30 candidate chunks per query.
3. **Adjudicate the pool with an LLM judge.** For each candidate, ask a strong LLM whether the passage answers the query (fully / partially / not at all). Calibrate the judge against 50–100 hand-labeled items before trusting it at scale; target >85% agreement.
4. **Method B additionally.** For hand-written queries, also keyword-search the corpus using clinical terms the annotator expects in relevant passages, and manually review every hit. Worth the time on the ~100-item trusted set.

Final labels per query: seed positive(s) plus all chunks labeled fully or partially relevant by the adjudication pipeline.

**Completeness audit.**

Build a 30-query gold-standard subset (20 from Method A, 10 from Method B) labeled exhaustively: pool top-20 from 6+ retrievers, LLM-judge everything, hand-review all LLM-relevant labels plus a random sample of LLM-not-relevant labels. Compare retriever scores on this subset using (a) exhaustive labels vs (b) pipeline labels. If the gap is <2–3 pp on the primary metrics, pipeline labels are fit for purpose; if larger, expand the pool or improve the judge. Report the audit result alongside the main retrieval scores — it is the error bar on all retrieval numbers.

**Metric choice under incomplete labels.**

Metrics vary in how much they suffer when positives are missing. Lead with completeness-robust metrics; report the rest as secondary (see §3.1):

- Robust: hit rate @ k, MRR.
- Moderately sensitive: nDCG@k.
- Most sensitive: Recall@k, Precision@k.

**Versioning.**

The benchmark is tied to a specific corpus snapshot and chunking scheme. Treat `(corpus_version, chunking_scheme, queries, labels)` as a single versioned artifact. If the corpus is refreshed or chunking changes, chunk IDs change and the labeling pipeline must be re-run.

---

## 2. mamabench — end-to-end QA

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

## 3. mamaretrieval — retrieval benchmark

### 3.1 Retriever-only metrics

Run the deployed retriever (same embedding model, same chunker, same top-k) over the query set. Score against `relevant_chunk_ids`:

- **Recall@k, Precision@k** at k = 1, 3, 5, 10.
- **MRR** (Mean Reciprocal Rank).
- **nDCG@k**.
- **Hit rate** — fraction of queries where any relevant chunk is retrieved.

Report separately on the LLM-generated query set and the hand-written query set. Large gaps between the two are a signal that LLM-generated queries are unrealistic.


### 3.2 Retriever comparators (optional)

If retrieval performance is a concern, compare the deployed retriever against:

- BM25 baseline
- A different embedding model (e.g., E5, BGE-M3, medical-domain embedding)
- Hybrid (BM25 + dense)
- Re-ranker added on top

Same query set, same corpus, same k. Report the same metrics.

---

## 4. Gemma 4 Behaviour Diagnostics

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

A publishable deployment story typically reads:

> "Gemma 4 E4B without retrieval scores X% on mamabench. With RAG over the WHO / Tanzania MOH guideline corpus, it reaches Y%, closing most of the gap to GPT-5 without retrieval (Z%). GPT-5 + our RAG reaches W%, indicating further headroom if compute were available, but the deployable Gemma + RAG configuration already achieves clinically useful performance at edge-device cost. Retrieval metrics on mamaretrieval show the pipeline hits Recall@5 of R and MRR of M over the guideline corpus. Under oracle retrieval, Gemma reaches O%, indicating the retriever — not the generator — is the primary bottleneck / is already close to saturating the generator. The model is stable under paraphrasing and re-sampling (variance < V), and on the local-guideline-aligned safety subset outperforms all no-RAG baselines, including frontier models."

That sentence is the artifact this plan is designed to produce, honestly.
