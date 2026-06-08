# MAMAI evaluation + mamabench + mamaretrieval

Evaluation framework for a Gemma 4 E4B + RAG medical-advice chatbot deployed for nurses and midwives in Zanzibar. Domain: OBGYN, neonatal / infant, reproductive health.

This repo is the **design / methodology** layer. Built artifacts and live results live in sibling repos — see *Current results* below.

## Documents

| File | Description |
|---|---|
| [`mamai-quality-evaluation.md`](mamai-quality-evaluation.md) | Quality evaluation — retrieval, generation faithfulness, and end-to-end QA |
| [`mamai-quality-evaluation-minimal.md`](mamai-quality-evaluation-minimal.md) | Executive summary of the quality evaluation (short read) |
| [`mamai-latency-evaluation.md`](mamai-latency-evaluation.md) | Latency evaluation — retrieval, generation, and end-to-end latency on deployment hardware |
| [`mamabench.md`](mamabench.md) | QA benchmark spec (MCQ + open-ended + safety sets) |
| [`mamaretrieval.md`](mamaretrieval.md) | Retrieval benchmark spec (query construction, labeling pipeline, metrics) |
| [`mamai-finetuning-plan.md`](mamai-finetuning-plan.md) | RAG-aware fine-tuning plan (**gated; not currently triggered** — oracle faithfulness ~0.3%) |
| [`rag-small-vs-large-literature.md`](rag-small-vs-large-literature.md) | Literature review — can a small model + RAG match a frontier model without RAG? |
| [`next-steps-2026-06.md`](next-steps-2026-06.md) | Forward-looking plan for June (the final month) — per-repo next steps and cross-cutting decisions |

## Evaluation approach

Bottom-up: validate the **retriever** (mamaretrieval) → validate the **generator** in isolation → measure the **end-to-end system** (mamabench).

## Current results

The documents in this repo are *specs*. Built artifacts and live numbers live here:

| Surface | Where to find it |
|---|---|
| Retrieval benchmark (data + judgments) | [`nmrenyi/mamaretrieval`](https://huggingface.co/datasets/nmrenyi/mamaretrieval) on HF (v0.2.0) · [`mamaretrieval`](https://github.com/nmrenyi/mamaretrieval) code + audit reports |
| QA benchmark (MCQ + open-ended + judge calibration) | [`nmrenyi/mamabench`](https://huggingface.co/datasets/nmrenyi/mamabench) on HF (v0.2.1) · [`mamabench`](https://github.com/nmrenyi/mamabench) code |
| End-to-end evaluation harness + reports | [`mamai-eval`](https://github.com/nmrenyi/mamai-eval) — `configs/config-v0.2.0/reports/` |
| Generator faithfulness pipeline + results | `mamai-eval` branch `feat/faithfulness-eval` — `docs/faithfulness-eval-v0.2.0.md` |
| On-device latency benchmark + report | [`mamai`](https://github.com/nmrenyi/mamai) — `evaluation/reports/latency_report_v2.md` |
| Deployed Android app | [`mamai`](https://github.com/nmrenyi/mamai) |
| Guideline corpus (extraction, chunking, embeddings) | [`mamai-medical-guidelines`](https://github.com/nmrenyi/mamai-medical-guidelines) |
| Tech report (in progress) | [`mamai-report`](https://github.com/nmrenyi/mamai-report) |
