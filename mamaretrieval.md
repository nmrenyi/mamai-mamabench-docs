# mamaretrieval

*A retrieval benchmark of `(query, relevant_chunk_ids)` pairs built from the production RAG corpus.*

*Used for retriever scoring and generator-behaviour diagnostics in the mamai evaluation.*

---

## Corpus

The production RAG corpus chunked under a fixed scheme; each chunk has a stable ID.

## Query sources

Two complementary methods.

**LLM-generated (bulk):** for each chunk, prompt a strong LLM to generate 2–3 realistic nurse/midwife questions answered by that chunk. The chunk becomes a seed positive. Target: 1,000–3,000 queries.

**Hand-written topic queries (small, trusted):** ~100 queries written by hand against the scope (PPH management, pre-eclampsia thresholds, neonatal resuscitation, danger signs, PMTCT, anaemia, malaria in pregnancy, etc.). Seed positives identified by reading the corpus.

## Relevance labeling

The core challenge is that each query may be answered by multiple chunks, not just one. Missing positives bias retriever scores pessimistically and non-uniformly across retrievers. Complete labels cannot be guaranteed but can be approximated via TREC-style pooling plus systematic expansion.

Pipeline per query:

1. **Seed positive.** Method A: the source chunk the query was generated from. Method B: positives identified by the annotator while writing the query.
2. **Pool candidates across retrievers.** Run the query through 3–4 strong retrievers (BM25, at least one alternative embedding model, optionally hybrid). Union the top-10 results, dedupe. Yields ~20–30 candidate chunks per query.
3. **Adjudicate the pool with an LLM judge.** For each candidate, ask a strong LLM whether the passage answers the query (fully / partially / not at all). Calibrate the judge against 50–100 hand-labeled items before trusting it at scale; target >85% agreement.
4. **Method B additionally.** For hand-written queries, also keyword-search the corpus using clinical terms the annotator expects in relevant passages, and manually review every hit. Worth the time on the ~100-item trusted set.

Final labels per query: seed positive(s) plus all chunks labeled fully or partially relevant by the adjudication pipeline.

## Completeness audit

Build a 30-query gold-standard subset (20 from Method A, 10 from Method B) labeled exhaustively: pool top-20 from 6+ retrievers, LLM-judge everything, hand-review all LLM-relevant labels plus a random sample of LLM-not-relevant labels. Compare retriever scores on this subset using (a) exhaustive labels vs (b) pipeline labels. If the gap is <2–3 pp on the primary metrics, pipeline labels are fit for purpose; if larger, expand the pool or improve the judge. Report the audit result alongside the main retrieval scores — it is the error bar on all retrieval numbers.

## Metric choice under incomplete labels

Metrics vary in how much they suffer when positives are missing. Lead with completeness-robust metrics; report the rest as secondary:

- **Robust:** hit rate @ k, MRR.
- **Moderately sensitive:** nDCG@k.
- **Most sensitive:** Recall@k, Precision@k.

## Versioning

The benchmark is tied to a specific corpus snapshot and chunking scheme. Treat `(corpus_version, chunking_scheme, queries, labels)` as a single versioned artifact. If the corpus is refreshed or chunking changes, chunk IDs change and the labeling pipeline must be re-run.
