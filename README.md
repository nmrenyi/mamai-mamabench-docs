# MAMAI evaluation + mamabench + mamaretrieval

Evaluation framework for a Gemma 4 E4B + RAG medical-advice chatbot deployed for nurses and midwives in Zanzibar. Domain: OBGYN, neonatal / infant, reproductive health.

## Documents

| File | Description |
|---|---|
| [`mamai-quality-evaluation.md`](mamai-quality-evaluation.md) | Quality evaluation — retrieval, generation faithfulness, and end-to-end QA |
| [`mamai-latency-evaluation.md`](mamai-latency-evaluation.md) | Latency evaluation — retrieval, generation, and end-to-end latency on deployment hardware |
| [`mamabench.md`](mamabench.md) | QA benchmark spec (MCQ + open-ended + safety sets) |
| [`mamaretrieval.md`](mamaretrieval.md) | Retrieval benchmark spec (query construction, labeling pipeline, metrics) |

## Evaluation approach

Bottom-up: validate the **retriever** (mamaretrieval) → validate the **generator** in isolation → measure the **end-to-end system** (mamabench).

> **Note:** A thorough latency evaluation may require deeper knowledge of the Gemma 4 model architecture — specifically how the model loads weights into memory and how it manages the KV cache across inference calls. To be considered in a later iteration of the latency evaluation plan.
