# MAMAI — Quality Evaluation (Minimal)

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

Run the deployed retriever (same embedding model, same chunker, same top-k) over the mamaretrieval dataset. Score against `relevant_chunk_ids`:

- **Recall@k, Precision@k** at k = 1, 3, 5, 10.
- **MRR** (Mean Reciprocal Rank).
- **nDCG@k**.
- **Hit rate** — fraction of queries where any relevant chunk is retrieved.

### 2.2 Retriever comparators

Compare the deployed retriever against:

- BM25 baseline
- A different embedding model (e.g., E5, BGE-M3, medical-domain embedding)
- Hybrid (BM25 + dense)
- Re-ranker added on top
- Advanced retriever models (e.g., LateOn)

Same query set, same corpus, same k. Report the same metrics.

---

## 3. Generator evaluation — LLM (Gemma 4) behaviour

All evaluations use oracle context from mamaretrieval.

### 3.1 Faithfulness / groundedness

Given the oracle retrieved context, measure what fraction of generated sentences are supported by the context using **MiniCheck**.

**Calibration.** Run a frontier model (Claude / GPT-4) on a random subset to validate MiniCheck scores. If both agree (Spearman ρ > 0.7), MiniCheck scores are trusted at scale.

### 3.2 Stability

Measured through MiniCheck claim-support rate under three probes:

- Prompt sensitivity
- Run-to-run variance
- Greedy vs sampled

### 3.3 Citation functionality check

When the model cites a guideline, verify that the cited source exists in the corpus and actually says what the model claims.

---

## 4. End-to-end evaluation — mamabench

### 4.1 Evaluation matrix

| Model | No RAG | + RAG (guideline corpus) |
|---|---|---|
| **Gemma 4 E4B** (deployment target) | baseline capability | **deployed system** |
| Medical fine-tuned (MedGemma 4B) | fine-tuning vs. RAG | fine-tuning + RAG |
| Frontier model (GPT-5 / Claude / Gemini 2.5) | capability ceiling | overall ceiling |

Key comparisons:

1. **Gemma-no-RAG → Gemma-RAG:** value added by retrieval.
2. **Gemma-RAG → MedGemma-no-RAG:** fine-tuning vs. RAG at the same 4B edge-device scale.
3. **Gemma-RAG → MedGemma-RAG:** performance gap when both use RAG; also compare RAG improvement ratio (Gemma-RAG gain vs MedGemma-RAG gain) to see which model benefits more from retrieval.
4. **Gemma-RAG → Frontier-no-RAG:** can a 4B edge model with guideline retrieval match a frontier model from pretraining alone?
5. **Frontier-no-RAG → Frontier-RAG:** does the corpus add information beyond frontier pretraining?

**Methodological constraint.** Keep the RAG pipeline identical across all "+ RAG" rows: same retriever, same embedding model, same chunking, same top-k, same prompt template. Only the generator changes.

**Metrics.**

- MCQ: accuracy, calibration (Brier score, ECE).
- Open-ended: HealthBench-style rubric scores (factuality, reasoning, harm, omission, guideline adherence).
- Safety: pass rate on EquityMedQA / FairMedQA items, robustness under MedEqualQA / MedFuzz perturbations.
