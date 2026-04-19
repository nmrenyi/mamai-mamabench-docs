# MAMAI evaluation + mamabench + mamaretrieval

Evaluation framework for a Gemma 4 E4B + RAG medical-advice chatbot deployed for nurses and midwives in Zanzibar. Domain: OBGYN, neonatal / infant, reproductive health.

## Documents

| File | Description |
|---|---|
| [`mamai-quality-evaluation.md`](mamai-quality-evaluation.md) | Quality evaluation — retrieval, generation faithfulness, and end-to-end QA |
| [`mamabench.md`](mamabench.md) | QA benchmark spec (MCQ + open-ended + safety sets) |
| [`mamaretrieval.md`](mamaretrieval.md) | Retrieval benchmark spec (query construction, labeling pipeline, metrics) |

## Evaluation approach

Bottom-up: validate the **retriever** (mamaretrieval) → validate the **generator** in isolation → measure the **end-to-end system** (mamabench).
