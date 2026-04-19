# mamabench

Evaluation framework for a Gemma 4 E4B + RAG medical-advice chatbot deployed for nurses and midwives in Zanzibar. Domain: OBGYN, neonatal / infant, reproductive health.

## Documents

| File | Description |
|---|---|
| [`mamai-evaluation-plan.md`](mamai-evaluation-plan.md) | Full evaluation plan — methodology, metrics, and narrative target |
| [`mamabench.md`](mamabench.md) | QA benchmark spec (MCQ + open-ended + safety sets) |
| [`mamaretrieval.md`](mamaretrieval.md) | Retrieval benchmark spec (query construction, labeling pipeline, metrics) |

## Evaluation approach

Bottom-up: validate the **retriever** (mamaretrieval) → validate the **generator** in isolation → measure the **end-to-end system** (mamabench).
