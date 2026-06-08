# MAMAI — Next Steps (June, the final month)

*Written 2026-06-08. Marks the pivot from benchmark construction to system improvement, per the week-13 weekly report.*

The past two months built the instruments: mamaretrieval (v0.2.0, Tier 3, 230k labelled pairs), mamabench (v0.2.1, MCQ + open-ended + judge-calibration set), the faithfulness pipeline (oracle → Gemma → Lynx, two passes, calibrated true-hallucination ≈ 0.3% at v0.1.0), and the on-device-vs-cluster calibration. Most of those instruments now have a result.

This month is about **using them to improve the system**, not adding more measurement surface. There are two genuinely high-leverage improvement levers: (1) corpus expansion — the ICM demo with Leah surfaced corpus coverage as the most visible gap; (2) retriever upgrade — gecko sits ~25 pp below voyage / octen / lateon on weighted precision at full scale. Everything else is finishing what's already in flight (open-ended judge, faithfulness calibration, safety set) or starting the report.

---

## Cross-cutting decisions to make first

These three decisions unblock work in multiple repos and shouldn't be made in isolation per-track.

1. **Consolidate the closed-source judge.** Two tracks each want a closed-source judge: the open-ended rubric (rubric track, currently testing gpt-5-mini — not materially better than gpt-oss-120b, +1.6 pp on not-met agreement) and the faithfulness calibration (currently blocked after gpt-oss-120b failed both gates with 75/100 calibration agreement vs ≥90 required, and 11.20% vs Claude 0.33% drift). Both should converge on the same tier. Likely escalation to GPT-5.5 — file one budget ask covering both tracks rather than two parallel ones.
2. **Retriever improvement — pursue *two* tracks in parallel, not one.**
   - **Track A — query rewriting before retrieval (priority, [`mamai#62`](https://github.com/nmrenyi/mamai/issues/62)).** Nurse queries are short and colloquial ("baby not breathing", "bleeding after birth"); the corpus is in clinical-guideline vocabulary. Rewriting closes the gap *without* changing the embedding model. Run the 3-step offline ablation in the issue first (rewrite + HyDE + multi-query, all measured against `kenya_vignettes` retrieval + downstream judge scores) before any on-device implementation. Strictly cheaper than swapping the retriever and likely complementary.
   - **Track B — embedding-model swap.** Voyage / octen / lateon are 25 pp ahead of gecko on weighted precision at full scale (Tier 2, n=3,185). Voyage is closed-weight; octen and lateon are open. Decide before re-running any expensive +RAG numbers — every retriever change invalidates the RAG context bundle and forces re-precompute.

   Track A is the higher-priority experiment because it ships even if Track B doesn't (and it informs Track B by isolating "retriever quality" from "query/corpus vocabulary mismatch").
3. **Corpus expansion as a project-level priority.** Leah's ICM demo finding (week-13 report) makes corpus coverage the single most visible deployment gap. The pipeline lives in [`mamai-medical-guidelines#3`](https://github.com/nmrenyi/mamai-medical-guidelines/issues/3) (systematic PubMed/PMC collection). Propagates to chunk IDs, RAG bundle, and faithfulness oracle. Worth scoping with Leah and Trevor before starting.

---

## Per-repo next steps

### `mamai-eval` (main + `feat/judge-responses-api-20260605`)

The open-ended ±RAG report has been blocked here since 2026-05-21. Finishing this is the single highest-leverage item in the project.

1. **Commit the uncommitted gpt-5-mini smoke results** sitting in `calibration/judge_validation/{reports,verdicts}/gpt-5-mini/`.
2. **Escalate the judge tier** per the cross-cutting decision. Run Phase A calibration on the chosen tier against the v0.2.1 OBGYN physician set (6,853 triples).
3. **Run Phase B production rescore** on the full 38,308 HealthBench rubric responses (±RAG), once the judge is validated.
4. **Write the open-ended ±RAG report**, mirroring `mcq-rag-effect-20260520.md`. This is the deployment-claim report.
5. **Add open-ended on-device vs cluster calibration** (carry-over from week 11). MCQ calibration showed +2.7 pp delta within noise; open-ended is more deployment-realistic and may behave differently.
6. Merge `feat/judge-responses-api-20260605` once Phase B lands.

### `mamai-eval` worktree `feat/faithfulness-eval` (`mamai-eval-faithfulness/`)

Pipeline is built and run twice. One closed-source run away from a publishable headline.

1. **Re-run v0.2.0 calibration + FAIL categorization** with the closed-source judge from the cross-cutting decision. Produces the final calibrated true-hallucination rate (raw 94.18% PASS; v0.1.0 calibrated to ≈ 0.3% true hallucination).
2. **Design and run the real-retrieval faithfulness probe.** Oracle-context faithfulness is solved. The missing measurement is faithfulness when retrieval is the actual deployed gecko top-3 — this is the only experiment that can fire the fine-tuning plan's decision gate.
3. **Optional: stricter `score = 6` oracle as sensitivity check** (deferred from v0.1.0).
4. Merge PR #2 once v0.2.0 calibration lands.

### `mamaretrieval`

The benchmark is at v0.2.0 and stable. Open items are about the *deployed retriever* (gecko), not the benchmark itself.

1. **Fix the 6 self-contradictory oracle contexts** (issue #18) flagged by the faithfulness track. Cheap; makes faithfulness numbers cleaner.
2. **Execute the retriever decision** (swap vs investigate). If swap, validate the chosen model on the v0.2.0 test set, then re-run downstream RAG precomputes in `mamai-eval`.
3. **Noisy-query probe** (carry-over from week 10 TODO, still pending). Generate typo / broken-English variants of the existing 3,185 queries; re-score the 6 retrievers; report robustness deltas. One additional release, then move on.

### `mamai-medical-guidelines`

This is where the **highest-leverage improvement work happens this month**, per Leah's ICM demo finding.

1. **Corpus expansion sprint.** Scope with Leah which sources are missing (the demo surfaced specific gaps — capture which queries the demo couldn't answer). Prioritise OBGYN / neonatal / Zanzibar-specific.
2. **Re-extract + re-chunk** the new sources with the existing marker-pdf pipeline (refactored in week 5).
3. **Re-embed** with whichever retriever the cross-cutting decision lands on. New chunk IDs propagate downstream.
4. **Bump corpus version** and update the manifest. Downstream consumers (mamaretrieval, mamabench RAG, mamai app RAG bundle) pin to it.

### `mamabench`

1. **Build the safety set for v0.3.** Source survey is committed (2026-05-19); the implementation has not started. This was on the TODO list for both week 10 and week 12.
2. **Ship v0.2.2** noting the validated judge model in the dataset card, after the rubric judge lands.

### `mamai` (the Android app)

The big item is the query-rewriting investigation; everything else is downstream of the retriever / corpus decisions.

1. **Query rewriting before retrieval — offline ablation ([`#62`](https://github.com/nmrenyi/mamai/issues/62)).** Priority item. Run the 3-step plan from the issue, all in `mamai-eval` / `evaluation/` without touching app code:
   1. Generate rewrites with Gemma 4 E4B on `kenya_vignettes` queries; embed both original and rewrite; compare top-3 retrieval + downstream judge scores.
   2. Compare against cheaper alternatives in the same ablation: template-based expansion, HyDE (embed a one-sentence hypothetical answer), multi-query (union of top-3 from N variants).
   3. Only if a clear winner survives an adversarial safety review on `kenya_vignettes`, scope the on-device implementation (`rewriteQuery()` step in `RagPipeline.generateResponse()`, latency measurement, runtime toggle).
   Deployment model assumption: **Gemma 4 E4B stays** as both the rewrite model and the answer model (newer, better forward compatibility — see [`mamai#48`](https://github.com/nmrenyi/mamai/issues/48) for the gemma3n alternative, parked).
2. **Rebuild the RAG bundle** to the new corpus version + (possibly) new embedding model. Bump bundle version, sign, ship.
3. **Optional: extend `BenchmarkForegroundService` from MCQ to open-ended**, so the open-ended ±RAG report can include a device row. Low priority unless device-level open-ended numbers are needed for the report.

### `mamai-mamabench-docs`

Stale. Pure cleanup.

1. **Commit the local changes** — `README.md` is modified; `mamai-finetuning-plan.md` is untracked. Both have sat there since before 2026-05-25.
2. **Update `mamai-quality-evaluation.md` §3.1** to reflect the actual built pipeline: Lynx-70B (not MiniCheck) as primary, with the judge-selection journey (MiniCheck → Qwen3 rejected for circularity → Lynx) and the calibrated ≈ 0.3% true-hallucination headline.
3. **Mark `mamai-finetuning-plan.md` as "gated — not triggered"** in the README, citing the oracle-faithfulness result. The plan stays shelved until the real-retrieval faithfulness probe (see faithfulness §2) shows a collapse.

### `mamai-report`

The final deliverable. Currently just a LaTeX scaffold (2 commits, last touch 2026-05-17). With one month of internship remaining, this needs to start.

1. **Start drafting.** These sections can be populated immediately from existing artifacts:
   - Methodology + system overview (from `mamai-mamabench-docs/README.md`).
   - Retrieval benchmark + Tier 3 results (`mamaretrieval/AUDIT_REPORT_v2.md`).
   - Faithfulness pipeline + Lynx + calibration result (`mamai-eval-faithfulness/docs/faithfulness-eval-v0.2.0.md`).
   - Latency + FP16/FP32 GPU finding (`mamai/evaluation/reports/latency_report_v2.md`).
   - On-device vs cluster MCQ calibration (`mamai-eval/configs/config-v0.2.0/reports/calibration-mcq-20260519.md`).
2. **Leave open-ended ±RAG + safety as the last sections** — both are blocked on items above and shouldn't gate the rest of the writing.
3. **Decide the report's narrative target.** The interesting story is no longer "we built benchmarks" — it's: oracle-context faithfulness is essentially solved (~0.3%), the deployed retriever is the weakest link (25 pp behind frontier), and the deployment-blocker is corpus coverage. Frame around that.

---

## Suggested ordering this month

A rough sequence that minimises waiting on cross-cutting decisions.

| Week | Focus |
|---|---|
| W1 | Cross-cutting decisions (judge tier + retriever tracks). File the consolidated budget. Commit stale docs in `mamai-mamabench-docs`. Start corpus expansion scoping with Leah. **Kick off query-rewriting offline ablation ([`mamai#62`](https://github.com/nmrenyi/mamai/issues/62))** — Track A of the retriever improvement work. Begin drafting methodology + retrieval sections of `mamai-report`. |
| W2 | Run Phase A calibration on chosen judge. Re-run faithfulness v0.2.0 calibration. **Compare query rewriting / HyDE / multi-query results on `kenya_vignettes`**; decide whether to scope on-device implementation. Execute embedding-model swap decision (Track B). Start noisy-query probe. Continue corpus expansion. Continue `mamai-report` writing. |
| W3 | Run Phase B production rescore (open-ended HealthBench, ±RAG). Re-precompute RAG contexts on new corpus + (possibly) new retriever + query-rewriting variant if a winner emerged. Run real-retrieval faithfulness probe. Build safety set for mamabench v0.3. |
| W4 | Write the open-ended ±RAG report. Run open-ended on-device vs cluster calibration. Finalise `mamai-report` (including open-ended + safety sections). |

This sequence assumes the budget ask clears in W1. If it slips, faithfulness v0.2.0 calibration and open-ended Phase B both slide accordingly.

---

## What is explicitly *not* on this list

- **Executing the RAG-aware fine-tuning plan.** Oracle-context faithfulness ≈ 0.3% true hallucination — the dominant failure mode (Pandey "irrelevant generation") is essentially absent. The plan stays gated. Only the real-retrieval faithfulness probe (faithfulness §2) can flip this decision.
- **More benchmark construction beyond noisy queries + safety.** This month is for system improvement (per the week-13 weekly report); further benchmark surface comes after the report ships, if at all.
- **Agentic tool calling on Gemma 4** (week 6 TODO). Out of scope for this month — revisit only if the report ships ahead of schedule.

---

## References

- Week 13 weekly report (2026-06-08) — judge-calibration update + ICM presentation.
- Week 11 weekly report (2026-05-25) — MCQ ±RAG result, mamaretrieval Tier 3 release, faithfulness pipeline.
- `mamai-eval-faithfulness/docs/faithfulness-eval-v0.2.0.md` — full faithfulness methodology + result.
- `mamai-eval/calibration/judge_validation/reports/` — bake-off reports (gpt-oss-120b, Nemotron, Maverick, gpt-5-mini).
- `mamaretrieval/AUDIT_REPORT_v2.md` — Tier 2 + Tier 3 retrieval scoreboards.
- `mamai-mamabench-docs/mamai-finetuning-plan.md` — gated RAFT plan (uncommitted; commit before referencing externally).
