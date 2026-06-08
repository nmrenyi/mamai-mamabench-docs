# MAMAI — Next Steps (June, the final month)

*Written 2026-06-08. Marks the pivot from benchmark construction to system improvement, per the week-13 weekly report.*

The past two months built the instruments: mamaretrieval (v0.2.0, Tier 3, 230k labelled pairs), mamabench (v0.2.1, MCQ + open-ended + judge-calibration set), the faithfulness pipeline (oracle → Gemma → Lynx, two passes, calibrated true-hallucination ≈ 0.3% at v0.1.0), and the on-device-vs-cluster calibration. Most of those instruments now have a result.

This month is about **using them to improve the system**, not adding more measurement surface. There are two genuinely high-leverage improvement levers: (1) corpus expansion — the ICM demo with Leah surfaced corpus coverage as the most visible gap; (2) retriever upgrade — gecko sits ~25 pp below voyage / octen / lateon on weighted precision at full scale. Everything else is finishing what's already in flight (open-ended judge, faithfulness calibration, safety set) or starting the report.

**Hard anchor: internship ends 2026-06-30** (a Tuesday). This gives ~17 weekdays from 2026-06-08, with W4 effectively truncated to 2 days — the schedule below accounts for this.

---

## Cross-cutting decisions to make first

These three decisions unblock work in multiple repos and shouldn't be made in isolation per-track.

1. **Consolidate the closed-source judge.** Two tracks each want a closed-source judge: the open-ended rubric (rubric track, currently testing gpt-5-mini — not materially better than gpt-oss-120b, +1.6 pp on not-met agreement) and the faithfulness calibration (currently blocked after gpt-oss-120b failed both gates with 75/100 calibration agreement vs ≥90 required, and 11.20% vs Claude 0.33% drift). Both should converge on the same tier. Likely escalation to GPT-5.5 — file one budget ask covering both tracks rather than two parallel ones.
2. **Retriever improvement — two parallel tracks.**
   - **Track A (priority): query-side improvements** ([`mamai#62`](https://github.com/nmrenyi/mamai/issues/62)). Closes the user-vocabulary / corpus-vocabulary gap without swapping the embedding model. Techniques (rewriting / HyDE / multi-query) and eval-venue rationale: [`mamai-quality-evaluation.md` §2.3](mamai-quality-evaluation.md#23-query-side-improvements).
   - **Track B: embedding-model swap.** Voyage / octen / lateon are ~25 pp ahead of gecko at full scale (Tier 2, n=3,185). Decide before re-running any expensive +RAG numbers — a swap invalidates the RAG context bundle.

   Track A is higher priority because it ships even if Track B doesn't, and isolates retriever quality from vocabulary mismatch.
3. **Corpus expansion as a project-level priority.** Leah's ICM demo finding (week-13 report) makes corpus coverage the single most visible deployment gap. The pipeline lives in [`mamai-medical-guidelines#3`](https://github.com/nmrenyi/mamai-medical-guidelines/issues/3) (systematic PubMed/PMC collection). Propagates to chunk IDs, RAG bundle, and faithfulness oracle. Worth scoping with Leah and Trevor before starting.

---

## Per-repo next steps

### `mamai-eval` (main + `feat/judge-responses-api-20260605`)

The open-ended ±RAG report has been blocked here since 2026-05-21. Finishing this is the single highest-leverage item in the project.

1. **Commit the uncommitted gpt-5-mini smoke results** sitting in `calibration/judge_validation/{reports,verdicts}/gpt-5-mini/`.
2. **Escalate the judge tier** per the cross-cutting decision. Run Phase A calibration on the chosen tier against the v0.2.1 OBGYN physician set (6,853 triples).
3. **Resolve [`mamai-eval#1`](https://github.com/nmrenyi/mamai-eval/issues/1) — the `generation.max_tokens=2048` question — before Phase B.** Open since 2026-05-19. The answer affects whether the open-ended numbers Phase B produces are final or need re-running; cheaper to decide before spending the production rescore than after.
4. **Run Phase B production rescore** on the full 38,308 HealthBench rubric responses (±RAG), once the judge is validated and `max_tokens` is settled.
5. **Write the open-ended ±RAG report**, mirroring `mcq-rag-effect-20260520.md`. The goal is to characterise the RAG effect on open-ended (real-world-shaped) queries — whether it helps or hurts on this distribution, by how much, on which categories. No pre-committed deployment criterion this month; the report informs the deploy / redesign / drop-RAG decision rather than gating it.
6. **Add open-ended on-device vs cluster calibration** (carry-over from week 11). MCQ calibration showed +2.7 pp delta within noise; open-ended is more deployment-realistic and may behave differently. Slips past 2026-06-30 if W3/W4 get tight.
7. Merge `feat/judge-responses-api-20260605` once Phase B lands.

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

1. **Query-side improvements — offline ablation ([`#62`](https://github.com/nmrenyi/mamai/issues/62)).** Priority. Run the 3-step plan from the issue (offline ablation first, on-device implementation gated on a safety review). Techniques and eval venue: [`mamai-quality-evaluation.md` §2.3](mamai-quality-evaluation.md#23-query-side-improvements). Deployment model stays **Gemma 4 E4B** for both rewrite + answer (gemma3n alternative parked, see [`mamai#48`](https://github.com/nmrenyi/mamai/issues/48)).
2. **Rebuild the RAG bundle** to the new corpus version + (possibly) new embedding model. Bump bundle version, sign, ship.
3. **Optional: extend `BenchmarkForegroundService` from MCQ to open-ended**, so the open-ended ±RAG report can include a device row. Low priority unless device-level open-ended numbers are needed for the report.

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

A rough sequence that minimises waiting on cross-cutting decisions. W4 is only 2 weekdays (Mon–Tue, 29–30 June), so report writing has to start in W3, not W4.

| Week | Dates | Focus |
|---|---|---|
| W1 | Jun 8–12 | Cross-cutting decisions (judge tier + retriever tracks). File the consolidated budget ask. Resolve [`mamai-eval#1`](https://github.com/nmrenyi/mamai-eval/issues/1) (`max_tokens` question). Start corpus expansion scoping with Leah. **Kick off query-rewriting offline ablation ([`mamai#62`](https://github.com/nmrenyi/mamai/issues/62))** — Track A. Begin drafting methodology + retrieval sections of `mamai-report`. |
| W2 | Jun 15–19 | Run Phase A calibration on chosen judge. Re-run faithfulness v0.2.0 calibration. **Compare query rewriting / HyDE / multi-query results on `kenya_vignettes`**; decide whether to scope on-device implementation. Execute embedding-model swap decision (Track B). Start noisy-query probe. Continue corpus expansion. Draft faithfulness + latency sections of `mamai-report`. |
| W3 | Jun 22–26 | Run Phase B production rescore (open-ended HealthBench, ±RAG). Re-precompute RAG contexts on new corpus + (possibly) new retriever + query-rewriting variant if a winner emerged. Run real-retrieval faithfulness probe. **Start writing the open-ended ±RAG report** in parallel with the run. |
| W4 | Jun 29–30 | **Finalise open-ended ±RAG report. Finalise `mamai-report` and hand off.** Anything else (safety set, on-device open-ended calibration) slips past 2026-06-30 and gets logged as post-internship TODO. |

This sequence assumes the budget ask clears in W1. If it slips, faithfulness v0.2.0 calibration and open-ended Phase B both slide accordingly — and given W4 is 2 days, **any W1 budget slip likely pushes the open-ended ±RAG report past 2026-06-30.** Flag this to Annie / Trevor early if W1 looks slow.

---

## What is explicitly *not* on this list

- **Executing the RAG-aware fine-tuning plan.** Oracle-context faithfulness ≈ 0.3% true hallucination — the dominant failure mode (Pandey "irrelevant generation") is essentially absent. The plan stays gated. Only the real-retrieval faithfulness probe (faithfulness §2) can flip this decision.
- **More benchmark construction beyond noisy queries + safety.** This month is for system improvement (per the week-13 weekly report); further benchmark surface comes after the report ships, if at all.
- **Agentic tool calling on Gemma 4** (week 6 TODO). Out of scope for this month — revisit only if the report ships ahead of schedule.
- **User testing in Zanzibar.** No longer pursued this internship cycle.
- **Manual review of low-safety responses ([`mamai#50`](https://github.com/nmrenyi/mamai/issues/50), P1).** Deferred — this is a pre-pilot-expansion deployment gate; since no pilot expansion is happening this month, it's logged as a post-internship TODO. Worth re-surfacing when whoever picks up the project starts moving toward a pilot.
- **Swahili evaluation ([`mamai#27`](https://github.com/nmrenyi/mamai/issues/27), [`#29`](https://github.com/nmrenyi/mamai/issues/29)).** No formal Swahili eval this month; logged as post-internship TODO. Genuine deployment-quality gap for the Zanzibar bilingual context.
- **HF Space web demo ([`mamai#63`](https://github.com/nmrenyi/mamai/issues/63)).** Recent issue. Useful for researcher self-serve access but not on the critical path; revisit if W3/W4 has slack.
- **`mamai#49` RAG injection-format ablations.** Contingency if query rewriting (Track A) alone does not close the RAG regression. The issue proposes 4 concrete variants (strip `Document N:` labels, add preamble, reorder, top-k vary). Not a W1–W2 priority; surface only if Track A results disappoint.

---

## References

- Week 13 weekly report (2026-06-08) — judge-calibration update + ICM presentation.
- Week 11 weekly report (2026-05-25) — MCQ ±RAG result, mamaretrieval Tier 3 release, faithfulness pipeline.
- `mamai-eval-faithfulness/docs/faithfulness-eval-v0.2.0.md` — full faithfulness methodology + result.
- `mamai-eval/calibration/judge_validation/reports/` — bake-off reports (gpt-oss-120b, Nemotron, Maverick, gpt-5-mini).
- `mamaretrieval/AUDIT_REPORT_v2.md` — Tier 2 + Tier 3 retrieval scoreboards.
- [`mamai-finetuning-plan.md`](mamai-finetuning-plan.md) — gated RAFT plan, not currently triggered.
