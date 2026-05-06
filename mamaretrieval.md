# mamaretrieval

*A retrieval benchmark of `(query, relevant_chunk_ids)` pairs built from the production RAG corpus.*

*Used for retriever scoring and generator-behaviour diagnostics in the mamai evaluation.*

---

## Corpus

The production RAG corpus; each chunk has a stable ID.

## Query sources

All queries are LLM-generated. For each chunk, prompt a strong LLM to generate 2–3 questions answered by that chunk; the chunk becomes the seed positive. Target: 1,000–3,000 queries.

To compensate for the absence of hand-written queries, the generation prompt applies three mitigations:

- **Synthesis questions:** generate 1 question per clinical topic area (not per chunk) that cannot be fully answered by any single passage, covering topics such as PPH management, pre-eclampsia thresholds, neonatal resuscitation, PMTCT, anaemia, malaria in pregnancy.
- **Nurse voice:** phrase questions as a nurse in Zanzibar would ask them — action-oriented, first-person, colloquial — rather than as exam questions.
- **Adversarial reformulations:** rephrase 10–20% of questions using local clinical abbreviations (PPH, MgSO4, PMTCT) rather than spelled-out terms.

**Residual caveat:** LLM-generated questions are phrased similarly to chunk text, which can inflate dense-retriever scores. Relative rankings across retrievers are more reliable than absolute numbers. Lead with MRR and hit rate rather than Recall@k.

> **Why no hand-written queries?** Hand-written queries were considered but skipped for resource reasons. What is lost: a trusted calibration anchor for the completeness audit, multi-chunk synthesis coverage, and harder natural-language phrasing that levels the playing field between dense and sparse retrievers. The three mitigations above partially close the gap.

## Relevance labeling

Each query may be answered by multiple chunks, not just one. Missing positives bias retriever scores pessimistically and non-uniformly. Labels are approximated via TREC-style pooling.

Pipeline per query:

1. **Seed positive.** The source chunk the query was generated from.
2. **Pool candidates across retrievers.** Run the query through 3–4 retrievers (BM25, at least one embedding model, optionally hybrid). Union the top-10 results, dedupe. Yields ~20–30 candidate chunks per query.
3. **Adjudicate with an LLM judge.** For each candidate, ask a strong LLM whether the passage answers the query (fully / partially / not at all). Calibrate the judge against 50–100 items before trusting it at scale; target >85% agreement.

Final labels per query: seed positive plus all chunks labeled fully or partially relevant.

## Completeness audit

Build a 30-query gold-standard subset labeled exhaustively:

1. Pool top-20 results from 6+ retrievers for each query.
2. LLM-judge every candidate chunk (fully / partially / not relevant).
3. Hand-review all LLM-relevant labels plus a random sample of LLM-not-relevant labels.

Compare retriever scores on this subset using (a) exhaustive labels vs (b) pipeline labels. If the gap is <2–3 pp on the primary metrics, pipeline labels are fit for purpose; if larger, expand the pool or improve the judge. Report the audit result alongside the main retrieval scores — it is the error bar on all retrieval numbers.

## Metric choice under incomplete labels

Lead with completeness-robust metrics; report the rest as secondary:

- **Robust:** hit rate@k, MRR.
- **Moderately sensitive:** nDCG@k.
- **Most sensitive:** Recall@k, Precision@k.

## Versioning

The benchmark is tied to a specific corpus version. Corpus version and chunking scheme are coupled in the corpus repository, so `(corpus_version, queries, labels)` is the versioned artifact. If the corpus is updated, chunk IDs change and the labeling pipeline must be re-run.
