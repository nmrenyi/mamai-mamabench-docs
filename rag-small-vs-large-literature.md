# Literature Review: Can Small LLMs + RAG Match Large LLMs Without RAG?

*Prepared for MAMAI project — Gemma 4 E4B + RAG over clinical guidelines, Zanzibar*

*Covers the specific comparison in §4.1 comparison 3: "Gemma-RAG → Frontier-no-RAG — the deployability headline."*

---

## Summary

The literature provides a nuanced answer: in many domain-specific settings, a small model with RAG can match or approach a larger model without RAG, but this is not universal. The outcome depends on (1) the task type — factual guideline lookup vs. complex clinical reasoning, (2) whether the small model has RAG-aware fine-tuning, and (3) the size of the frontier gap. The most consistent finding is that RAG benefits small models proportionally more than large ones, and that GPT-3.5-class models with strong RAG can reach GPT-4-level performance on medical knowledge tasks. The critical caveat for MAMAI: at 4B parameters without RAG-aware fine-tuning, standard RAG may be net-negative.

---

## Quantitative Expectation Baselines

| Condition | Approx. accuracy | Status | Source |
|---|---|---|---|
| GPT-3.5 + MedRAG | ~71.6% on MIRAGE | **Directly measured** | Xiong et al., ACL Findings 2024 |
| GPT-4 no RAG | ~73–79% | **Directly measured** | MIRAGE (73.4%); range across other benchmarks |
| GPT-4o no RAG (African medical QA) | ~79% | **Directly measured** | Olatunji et al., ACL 2025 (AfriMed-QA) |
| MedGemma 4B, medical fine-tuning, no RAG | ~64% on MedQA | **Directly measured** | MedGemma Technical Report, arXiv:2507.05201 |
| Gemma 4B, no RAG, no tuning | ~40–55% | **Extrapolated** | Inferred from Gemma-2B at 17% (AfriMed-QA), MedGemma-4B at 64% with fine-tuning, and 40–60% range for <10B models across multiple papers; no direct Gemma 4B vanilla measurement found |
| Gemma 4B + standard RAG, no RAG-aware tuning | Marginal gain or net-negative | **Extrapolated** | Pandey (arXiv:2603.11513, 2026) showed sub-7B standard RAG averages −3pp net; applied to 4B by extension — Gemma 4B specifically not tested |
| Gemma 4B + RAG + RAG-aware fine-tuning | ~70–75% on structured guideline QA | **Speculative** | No paper tests a 4B model with RAFT/Self-RAG-style tuning; upper bound inferred from RAFT-7B exceeding GPT-3.5+RAG and MedRAG's GPT-3.5+RAG ≈ 71.6% |

---

## Papers Supporting Small+RAG ≈ Large+No-RAG

### MedRAG / MIRAGE — Xiong et al., ACL Findings 2024

- **Citation:** Xiong et al., "Benchmarking Retrieval-Augmented Generation for Medicine," ACL 2024 Findings. arXiv:2402.13178.
- **Models:** GPT-4, GPT-3.5, Mixtral (8×7B), LLaMA2-70B, MEDITRON-70B, PMC-LLaMA-13B
- **Benchmark:** MIRAGE — 7,663 questions from MMLU-Med, MedQA-US, MedMCQA, PubMedQA, BioASQ
- **Key numbers:** GPT-3.5 no RAG: 60.69% → GPT-3.5 + MedRAG: 71.57% (+10.88pp). GPT-4 no RAG: 73.44%. Gap between GPT-3.5+RAG and GPT-4 no-RAG: **1.87pp**.
- **RAG benefit curve:** GPT-3.5 gained +17.9pp from RAG; MEDITRON-70B only +5.3pp — smaller models gain proportionally more.
- **Verdict:** The closest published analogy to MAMAI's architecture at a larger model size. Directly supports the "leapfrogging" claim.

### Self-RAG — Asai et al., ICLR 2024 (Oral, top 1%)

- **Citation:** Asai et al., "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection." arXiv:2310.11511.
- **Models:** Self-RAG 7B and 13B (fine-tuned LLaMA2) vs. ChatGPT (GPT-3.5)
- **Tasks:** PopQA, TriviaQA, PubHealth fact verification, long-form generation
- **Key finding:** Self-RAG 7B and 13B significantly outperformed ChatGPT on open-domain QA, reasoning, and fact verification.
- **Caveat:** Requires RAG-specific fine-tuning — the LLM is trained to emit special reflection tokens and decide adaptively when to retrieve. Not standard RAG.
- **Verdict:** Supports small+RAG > ChatGPT-class, with RAG-aware fine-tuning.

### RAFT — Zhang et al., 2024

- **Citation:** Zhang et al., "RAFT: Adapting Language Model to Domain Specific RAG." arXiv:2403.10131.
- **Models:** RAFT-7B (fine-tuned LLaMA-2-7B) vs. GPT-3.5 with and without RAG
- **Domains:** PubMed (medical), HotpotQA, HuggingFace/Torch Hub documentation
- **Key finding:** RAFT-7B outperformed GPT-3.5+RAG on most benchmarks. Gains as large as +35pp on HotpotQA and +76pp on Torch Hub. RAFT trains the model on (question, retrieved docs including distractors, CoT answer) triples, teaching distractor robustness.
- **Verdict:** Supports small+fine-tuned+RAG > GPT-3.5+RAG on domain-specific tasks.

### RadOncRAG — Thaker et al., JCO Clinical Cancer Informatics 2025

- **Citation:** Thaker et al., "RadOncRAG: A Novel Retrieval-Augmented Generation Framework Improves Large Language Model Benchmark Performance in Radiation Oncology." PubMed 41237352.
- **Models:** 15 LLMs including GPT-4-turbo, GPT-4o, GPT-4.1, GPT-5; knowledge base from Gunderson & Tepper + NCCN guidelines
- **Key numbers:** GPT-4-turbo no RAG: 73.1% → +RAG: 79.4%. GPT-4o no RAG: 77.3%. **GPT-4-turbo+RAG (79.4%) > GPT-4o no-RAG (77.3%)** — explicit leapfrogging.
- **Key quote:** "Older, more lightweight models derived the greatest benefit from RAG-augmentation."
- **Important caveat:** Reasoning models (o1, o3-mini, DeepSeek-R1) showed no improvement or slight decline with RAG. The leapfrogging effect applies to standard (non-reasoning) generation models.
- **Verdict:** The clearest published clinical example of an older/smaller model + RAG beating a newer/larger model without RAG.

### "To Memorize or to Retrieve" — Scaling Laws for RAG, arXiv:2604.00715, 2026

- **Models trained:** OLMo-2-based from 30M to 3B parameters
- **Key theoretical finding:** RAG benefit is largest for the smallest models and diminishes substantially as model size grows. Retrieval provides the greatest per-unit-compute gains where parametric capacity is smallest. Retrieval becomes efficient beyond a pretraining saturation threshold of ~D/N = 4.14 tokens per parameter.
- **Implication for MAMAI:** A 4B model trained on a moderate data budget sits in the range where retrieval provides meaningful gains. Provides theoretical grounding for the "small models benefit most" claim.

### Radiology Contrast Media — Bressem et al., npj Digital Medicine 2025

- **Citation:** Bressem et al., "Retrieval-augmented generation elevates local LLM quality in radiology contrast media consultation." PMC:12223273.
- **Models:** Llama 3.2-11B (local, ±RAG) vs. GPT-4o-mini, Gemini 2.0 Flash, Claude 3.5 Haiku (cloud, no RAG)
- **Guideline corpus:** ACR Manual on Contrast Media, ESUR guidelines, institutional protocols
- **Key finding:** Llama 3.2-11B + RAG: hallucination rate 8% → 0%; radiologist ranking 4.80 → 3.54. GPT-4o-mini no RAG: 3.21. **RAG-Llama approached GPT-4o-mini** while running locally with no cloud dependency.
- **Verdict:** Partially supports — small+RAG approaches but does not fully match frontier; eliminates hallucinations, which is clinically meaningful.

### Hepatology Guidelines — Sallout et al., npj Digital Medicine 2024

- **Citation:** Sallout et al., "Optimization of hepatological clinical guidelines interpretation by large language models." PMC:11039454.
- **Model:** GPT-4 Turbo only (no small-model comparison)
- **Key numbers:** GPT-4 Turbo no RAG: **43%** on EASL HCV guidelines. GPT-4 Turbo + structured RAG: **99%**.
- **Relevance:** RAG over authoritative structured guidelines is essential even for frontier models. Establishes that guideline-intensive clinical QA requires retrieval regardless of model size — making RAG a necessary foundation for MAMAI, not just a small-model compensator.

---

## Papers Qualifying or Contradicting the Claim

### "Can Small Language Models Use What They Retrieve?" — Pandey, arXiv:2603.11513, 2026

- **Models:** 360M to 8B (SmolLM2, Qwen2.5, Llama 3.1 architectures)
- **Key findings:**
  - At ≤7B without RAG-specific training, standard RAG averages **−3pp net** vs. no retrieval
  - With oracle retrieval (correct answer guaranteed in context), only 14.6% of otherwise unanswerable questions were rescued at 7B — **85% of retrieval effort wasted**
  - Dominant failure mode: "irrelevant generation" — models ignore retrieved passages entirely (61–100% of failures)
  - Robust context utilization appears to emerge consistently only above ~10B parameters
- **Implication for MAMAI:** At 4B, standard RAG without RAG-aware fine-tuning likely fails to yield the headline gain. RAG-aware instruction tuning (Self-RAG or RAFT-style) is likely necessary to make the architecture work.
- **Verdict:** The most important caveat in the literature for the MAMAI use case.

### RAG for 10 LLMs in Medical Fitness — Lim et al., npj Digital Medicine 2025

- **Citation:** Lim et al., "Retrieval augmented generation for 10 large language models." PMC:11971376.
- **Models:** GPT-3.5, GPT-4, GPT-4o, Llama2-7B/13B/70B, Llama3-8B/70B, Gemini-1.5-Pro, Claude-3-Opus
- **Guidelines:** 35 local hospital + 23 international anesthesia guidelines
- **Key numbers:** GPT-4 no RAG: 92.9%. Llama2-13B with RAG: <50% on complex ASA3 cases.
- **Verdict:** Where GPT-4 is near-ceiling (92.9%), the gap is too large for standard RAG on a 13B model to close.

### Neurology Guidelines — Muller et al., npj Digital Medicine 2025

- **Citation:** Muller et al., "Evaluating base and retrieval augmented LLMs for evidence based neurology." PMC:11880332.
- **Models:** GPT-4o, GPT-4 Turbo, GPT-4o-mini, LLaMA3-70B, Mixtral-8x7B, Gemini-1.5 Pro; RAG variants including online-RAG LLaMA3.1-Sonar (405B)
- **Key numbers:** Mixtral-8x7B no RAG: 25%. GPT-4o no RAG: 60%. Online-RAG LLaMA3.1-Sonar (405B): 67% — exceeds GPT-4 Turbo no RAG (44%) but not GPT-4o no RAG.
- **Key failure mode:** RAG systems performed worse on case-based than knowledge-based questions — directly relevant because MAMAI's nurses ask clinical management questions, not pure factual recall.
- **Verdict:** Mixed. Large+RAG clearly wins; no small model (<70B) with standard RAG matched GPT-4o without RAG.

### MRAG Biomedical Benchmark — Zhu et al., arXiv:2601.16503, 2026

- **Models:** GPT-4, GPT-3.5, Mixtral-8x22B, LlaMA-3-70B, Qwen2.5-72B, MEDITRON-70B, PMC-LLaMA-13B, Qwen2.5 (0.5B–32B)
- **Key finding:** "The scaling law of LLMs applies under RAG." Larger models gain more from RAG in absolute terms. PMC-LLaMA (13B) improved from 48.4% to 52.5% with RAG but remained far below GPT-4.
- **Note:** This study finds larger models gain more in absolute RAG benefit, while "To Memorize or to Retrieve" finds smaller models gain more per-unit-of-retrieved-data. The difference is in the metric (absolute accuracy gain vs. proportional gain relative to parametric baseline).
- **Verdict:** In biomedical RAG, scaling law still applies; the gap to frontier models persists even with retrieval.

### Rare Disease Diagnosis — He et al., JMIR 2025

- **Citation:** He et al., "Performance of ChatGPT-4o and Four Open-Source LLMs in Rare Disease Diagnosis." PMC:12192912.
- **Models:** ChatGPT-4o, Qwen2.5-7B (±RAG), Qwen2.5-72B, Llama variants
- **Key numbers:** ChatGPT-4o no RAG: 90.1%. Qwen2.5-7B no RAG: 47.9%. Qwen2.5-7B + RAG: 79.3% (+31.4pp).
- **Verdict:** Dramatic uplift from RAG, but a 10.8pp gap to GPT-4o remains.

### "Diminishing Returns" for Complex RAG Training — Ding et al., arXiv:2502.11400, 2025

- **Models:** 0.5B to 70B (Llama-2, Llama-3, Qwen1.5, Qwen2.5)
- **Key finding:** For larger models, sophisticated RAG training yields diminishing returns. The robustness benefit of complex document selection dropped from 14.65% (Llama-2) to 4.36% (Llama-3) on TriviaQA. By 70B, simple and sophisticated RAG pipelines converge to within ~2pp.
- **Implication:** Larger models are naturally more robust to noisy retrieval without specialized training; small models are not. This is why small models need RAFT/Self-RAG-style fine-tuning.

---

## Domain and Task-Type Breakdown

| Domain | Small+RAG vs. Frontier no-RAG | Key Paper |
|---|---|---|
| Medical QA (structured MCQ) | Approaches: ~1.9pp gap at GPT-3.5+RAG vs. GPT-4 | MedRAG, ACL 2024 |
| Radiation oncology (MCQ) | Leapfrogs: older+RAG > newer no-RAG | RadOncRAG, JCO 2025 |
| Clinical guidelines (hepatology, NICE) | RAG essential even for frontier models | Sallout 2024; arXiv:2510.02967 |
| Neurology guidelines (case-based) | Fails: case reasoning resists RAG | Muller et al., npj 2025 |
| Medical fitness / preop (complex) | Fails: 13B+RAG cannot match GPT-4 (92.9%) | Lim et al., npj 2025 |
| Rare disease diagnosis | Partially: 7B+RAG 79.3% vs. GPT-4o 90.1% | He et al., JMIR 2025 |
| Radiology contrast (local guidelines) | Partially: 11B+RAG approaches GPT-4o-mini | Bressem et al., npj 2025 |
| Legal entailment | Succeeds: 3B+prompting > GPT-4o-mini | Vaddi, arXiv 2026 |
| General-domain sub-7B (standard RAG) | Net-negative: −3pp without RAG-aware training | Pandey, arXiv 2026 |

---

## MAMAI-Specific Context Papers

### AfriMed-QA — Olatunji et al., ACL 2025 (Best Social Impact Award)

- **Models:** 30 LLMs from 2B to 540B+, no RAG conditions
- **Dataset:** ~15,000 questions from 60 African medical schools, 12 countries, 32 specialties
- **Key numbers:** Gemma-2B: 17%. Small models (<10B): 40–60%. GPT-4o: 79%.
- **Relevance:** The closest published benchmark to MAMAI's deployment setting. No RAG conditions were tested, but it establishes the parametric baseline gap that RAG must bridge. Biomedical-specific small models consistently underperformed general models of similar size.

### MedGemma Technical Report — Google DeepMind, arXiv:2507.05201, 2025

- **Key numbers:** MedGemma-4B (fine-tuned on medical data, no RAG): 64.4% on MedQA. MedGemma-27B: 87.7%. With LoRA fine-tuning, MedGemma-4B outperformed untuned GPT-4 on a vision classification task.
- **Relevance:** Establishes the Gemma 4B medical baseline. With medical fine-tuning alone (no RAG), the 4B model reaches ~64% on MedQA — meaningful, but below GPT-4. RAG is needed to push further.

### LLMs for Rwanda CHW — Ndayishimiye et al., Nature Health 2026

- **Models:** Gemini-2, GPT-4o, o3-mini, DeepSeek-R1, Meditron-70B vs. local Rwandan clinicians
- **Dataset:** 5,609 clinical questions from 101 community health workers in 4 districts
- **Key finding:** All 5 LLMs significantly outperformed local clinicians. Meditron-70B (medical fine-tuned) scored substantially lower than frontier models — aligning with the finding that domain-tuned smaller models often underperform large general models. No small (<10B) model or RAG conditions tested.
- **Relevance:** Establishes frontier model relevance for sub-Saharan African CHW contexts. MAMAI's Gemma 4B + RAG must aim to approach the Meditron-70B baseline at minimum.

### NICE Clinical Guidelines RAG — arXiv:2510.02967, 2025

- **System:** RAG over 10,195 chunks from 300 NICE guidelines
- **Key numbers:** RAG-enhanced O4-Mini: 99.5% faithfulness. Meditron3-8B (medical model, no RAG): 43% faithfulness. GPT-4.1 + RAG: 98.7%, reducing unsafe responses by 67%.
- **Relevance:** The most directly comparable published system to MAMAI (clinical guidelines in English, similar sources). RAG over structured guideline text produces near-perfect faithfulness even with a smaller model; parametric medical knowledge in an 8B model without RAG cannot approach this.

---

## Conditions Where Small+RAG Is Most Likely to Work

Based on the literature:

1. **Task type is factual lookup from structured guidelines** (retrieve-and-read), not multi-hop clinical reasoning across multiple patient variables
2. **The model has RAG-aware fine-tuning** — Self-RAG or RAFT-style training to condition on retrieved context rather than ignore it
3. **The retrieval corpus is high-quality and well-structured** — the hepatology study shows structured guidelines dramatically outperform raw PDFs even for frontier models
4. **The corpus covers the specific domain** — MAMAI's corpus (WHO, Tanzania MOH, Zanzibar MOH, ACOG, RCOG, NICE, FIGO, EmONC) is well-matched to the evaluation domain
5. **Questions are moderately complex** — not complex multi-step differential diagnosis requiring integration of many clinical variables

## Conditions Where the Claim Breaks Down

1. **Sub-7B without RAG-aware training:** standard RAG is likely net-negative (Pandey 2026)
2. **GPT-4 is near-ceiling on the task (>90%):** the gap is too large for standard RAG to close (Lim et al. 2025)
3. **Case-based clinical reasoning questions:** RAG systems underperform pure-knowledge lookup; nurses' management questions may fall in this category (Muller et al. 2025)
4. **Reasoning models as the backbone:** o1/o3-mini/DeepSeek-R1 show no improvement or slight decline with standard RAG (Thaker et al. 2025)

---

## Why Small Models Fail at RAG, and How Fine-Tuning Fixes It

*Deep-dive on the mechanisms behind context-utilization failure and the technical implementation of RAG-aware fine-tuning methods.*

---

### Why Small Models Fail to Use Retrieved Context

#### The Empirical Baseline

Pandey (arXiv:2603.11513, 2026) is the most controlled study to date. Five instruction-tuned models spanning 360M–8B (SmolLM2-360M, Qwen2.5-1.5B/3B/7B, Llama 3.1-8B) were tested under four retrieval conditions: no retrieval, BM25, dense retrieval (E5-large-v2), and oracle retrieval. All used 4-bit NF4 quantization.

The result can be expressed as a net trade-off:

> ΔEM_net = p_unk × ΔEM_unk + p_kn × ΔEM_kn

For a 7B model with dense retrieval: (0.815 × 8.0) − (0.185 × 51.2) = **−3.0pp net**. With oracle retrieval (correct answer guaranteed in context), only 14.6% of otherwise-unanswerable questions were rescued; known-question destruction was 41.6–100pp. The dominant failure across all sizes: irrelevant generation — the model ignores the retrieved passage entirely. The 360M model showed 100% irrelevant generation; even the 7B was at 61%.

#### Mechanism 1 — Position Bias and "Lost in the Middle"

Liu et al. (TACL 2024, arXiv:2307.03172) established a U-shaped attention pattern over long contexts: tokens at the beginning and end receive disproportionate attention; middle-context content is systematically underweighted. Two causes: (a) causal attention accumulates higher scores at early positions; (b) RoPE (rotary positional embedding) introduces long-term decay that diminishes attention scores for distant tokens. For small models with fewer attention heads, both effects are more severe. In a standard RAG prompt (system instruction → retrieved passage → question), the retrieved passage sits in the degraded middle zone.

#### Mechanism 2 — Context-Parametric Inversion (CPI) from Instruction Fine-Tuning

Shi et al. (arXiv:2410.10796, ICLR 2025) identified a critical training dynamic: during instruction fine-tuning, context reliance initially increases but then *gradually decreases* as training progresses, even as benchmark accuracy keeps rising. Observed across Llama, Mistral, and Pythia families on TULU, Alpaca, and Ultrachat datasets.

The cause: instruction fine-tuning corpora contain many examples where the context is consistent with what the model already knows parametrically. The model learns a shortcut — "if I know the answer, I don't need to read the context." This shortcut persists at deployment even when the context *should* override parametric priors. **Gemma 4B-IT is likely already suffering from CPI** — the parametric-dominance behavior Pandey documented is amplified in instruction-tuned variants relative to base models.

#### Mechanism 3 — FFN Parametric Dominance

ReDeEP (arXiv:2410.11414, 2024) provides the mechanistic picture using two internal metrics:

- **ECS (External Context Score):** cosine similarity between the mean-pooled hidden states of top-attended tokens and the generated token's hidden state. Higher = model is copying from retrieved context.
- **PKS (Parametric Knowledge Score):** Jensen-Shannon divergence before/after FFN processing. Higher = FFN is pushing the model toward its parametric vocabulary distribution.

Hallucination in RAG occurs when **FFN layers in later layers overemphasize parametric knowledge**, while **Copying Heads** (attention heads that preserve externally attended information) fail to counteract this push. Small, shallow models have less attention capacity to counteract the FFN parametric push — architecturally, the context loses the competition.

#### Quantization Compounds the Problem

All local models in Pandey were run in 4-bit NF4. The −3pp result already reflects 4-bit quantization. Separately, arXiv:2505.20276 (EMNLP 2025) found that 4-bit quantization degrades long-context task performance by up to 59% across Llama-3.1 and Qwen-2.5 models, while 8-bit quantization causes only ~0.8% degradation. RAG context-following is among the hardest tasks for 4-bit quantization to preserve. **Deploying Gemma 4B in NF4 at inference compounds an already-fragile context-utilization capacity.**

#### Cross-Model-Family Comparison

CK-PLUG (arXiv:2503.15888, 2025) tested LLaMA2-7B, LLaMA3-7B, Mistral-7B, and Qwen2.5-7B, finding Qwen2.5-7B had notably stronger parametric confidence than the others at the same scale. The medical fine-tuning study (PMC12292519, 2025) tested Gemma-2-9B, Phi-3.5-Mini, Llama-3.1-8B, Mistral-7B, and Qwen2.5-7B, finding Gemma-2 and Mistral showed mixed RAG results while Llama showed the most consistent RAG benefit. No published study has tested Gemma 4B specifically.

---

### Self-RAG — Technical Implementation

**Citation:** Asai et al., "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection." arXiv:2310.11511. ICLR 2024 Oral (top 1%).

#### Core Idea

A single LM is fine-tuned to interleave generation with special **reflection tokens** and retrieved passages. There is no external reward model at inference — the LM itself produces tokens that gate retrieval and evaluate quality. Two separate models are trained: a Critic (labeler) and a Generator (responder).

#### Reflection Token Taxonomy

Four token types are added to the model vocabulary:

| Token | Values | Signal |
|---|---|---|
| `[Retrieve]` | yes / no / continue | Should the model retrieve passages now? |
| `[IsRel]` | relevant / irrelevant | Does retrieved passage d help answer query x? |
| `[IsSup]` | fully / partially / no support | Are generated statements supported by d? |
| `[IsUse]` | 5 / 4 / 3 / 2 / 1 | Is the response useful overall? |

These are interleaved with generation: the model generates a segment, emits `[IsSup]` and `[IsUse]`, then decides via `[Retrieve]` whether to retrieve before the next segment. Adaptive retrieval is the key differentiator from standard RAG — retrieval is conditional, not mandatory.

#### Critic Training

A Critic model (same architecture) is trained to predict reflection tokens given (input, output) pairs. GPT-4 was used to label 4,000–20,000 examples per token type via type-specific few-shot prompts. The Critic achieves >90% agreement with GPT-4 predictions and is then used offline to annotate the Generator training corpus.

Training objective: `max_C E[log p_C(r | x, y)]`

#### Generator Training

**Training data:** 150,000 instruction-output pairs from OpenInstruct data (ShareGPT, GPT-4 Alpaca, OpenAssistant, FLAN) plus knowledge-intensive datasets (Natural Questions, FEVER, ASQA, ARC-Easy, OpenBookQA). The Critic annotates these with reflection tokens and retrieved passages.

**Loss function:** Standard next-token prediction on the augmented text. **Retrieved passage content is masked from the loss** — the model is trained to predict reflection tokens and generated text, not to reconstruct the retrieved passage.

**Inference:** Segment-level beam search (width = 2). Candidate scores combine generation probability and weighted reflection token scores:

> f(y_t, d, Critique) = p(y_t | x, d, y_{<t}) + Σ_G w^G · s_t^G

where `s_t^G` is the softmax probability over the most desirable value for each critique type (e.g., "fully supported" for `[IsSup]`). Weights `w^G` can be adjusted at inference to trade off behaviors.

**Compute:** 8× A100 40GB (7B model), 4× A100 80GB (13B model).

#### Limitations

- Requires GPT-4 API access for Critic data generation (~$50–200 depending on scale).
- Vocabulary modification (tokenizer resize + embedding matrix expand) is an implementation complexity.
- Standard RAG still outperforms Self-RAG on tasks requiring inference beyond direct passage extraction.
- **No Self-RAG 2 exists.** Open-RAG (EMNLP 2024, arXiv:2410.01782) and Tiny-Critic RAG (arXiv:2603.00846, 2026) are the closest functional successors.

---

### RAFT — Technical Implementation

**Citation:** Zhang et al., "RAFT: Adapting Language Model to Domain Specific RAG." arXiv:2403.10131, 2024.

#### Core Idea

RAFT is a supervised fine-tuning recipe for closed-domain RAG. Given a question and a mixed set of oracle (relevant) and distractor (irrelevant) documents, the model learns to: (a) identify the relevant document, (b) cite supporting evidence verbatim in a chain-of-thought, and (c) answer correctly while ignoring distractors.

#### Training Example Structure

**Input:**
```
Question: <query>
Document 1: <oracle document — contains the answer>
Document 2: <distractor>
Document 3: <distractor>
Document 4: <distractor>
Document 5: <distractor>
```

**Target output:**
```
##Reason: The document ##begin_quote## [exact verbatim passage] ##end_quote##
establishes that [reasoning chain].
##Answer: [final answer]
```

The `##begin_quote##` / `##end_quote##` markers serve the same purpose as Self-RAG's `[IsSup]` token but are embedded in answer text — no tokenizer modification needed, which is a significant practical advantage.

#### Training Data Construction

- **Oracle-to-distractor ratio:** 1 oracle + 4 distractor documents (k=5). Ablations tested more distractors; k=5 was optimal.
- **Distractor selection:** Top-k results from the deployed retriever, excluding the oracle chunk — simulates realistic noisy retrieval.
- **P% with oracle:** ~80% of examples include an oracle document; 20% contain only distractors, training the model to fall back on parametric knowledge or acknowledge uncertainty when retrieval fails.
- **CoT generation:** Answers are generated by prompting GPT-4 or GPT-3.5 with (question, oracle chunk) and requesting a chain-of-thought answer citing verbatim evidence.
- **Fine-tuning method:** The paper uses standard SFT on LLaMA-2-7B. Community implementations use QLoRA successfully; RAFT does not require full fine-tuning.

#### Key Results

RAFT-7B gains of +35pp on HotpotQA and +76pp on Torch Hub documentation vs. its no-RAG baseline, and outperforms GPT-3.5+RAG on most benchmarks. P% optimum varies across datasets (40–100%), requiring validation on held-out domain data.

#### Limitations

- Domain-specific: a model RAFT-tuned on WHO guidelines may regress on general medical QA outside those guidelines.
- Minimal gain on binary yes/no tasks (PubMedQA), where citing evidence provides limited benefit.
- No formal handling of unanswerable queries — must be designed separately.

---

### Other RAG-Aware Fine-Tuning Methods

#### RA-DIT (Meta, arXiv:2310.01352, ICLR 2024)

Retrieval-Augmented Dual Instruction Tuning. Two-stage: (1) LM fine-tuning on ~1.2M instruction examples with retrieved "background" prepended to each input; (2) retriever fine-tuning using an LM-supervised retrieval loss that aligns the query encoder to return passages the fine-tuned LM prefers. Simpler than Self-RAG (no reflection tokens), more complete than RAFT (tunes both LM and retriever). Primary model: LLaMA 65B.

#### Simple SFT on (Question, Retrieved Docs, Answer) Triples

Finetune-RAG (arXiv:2505.10792, 2025): fine-tuning Llama 3.1-8B on 1,653 examples of (question, factual doc, fictitious doc, answer) yielded **+21.2% accuracy in 20 training steps** on a fact-grounding task. The minimum-viable version of RAFT, without CoT citation markers.

#### Tiny-Critic RAG (arXiv:2603.00846, 2026)

A Qwen-1.7B model fine-tuned via LoRA (r=16) as a binary routing critic: "high semantic relevance → generate from context" vs. "contradictory distractors → use parametric fallback." Achieves routing F1 = 0.912 with 94.6% reduction in evaluation latency vs. large-LLM critics. Trainable on RTX 4090, 15 epochs, routing latency 42ms. **The most practical lightweight substitute for Self-RAG's full reflection token mechanism.**

#### Open-RAG (arXiv:2410.01782, EMNLP 2024)

Converts LLaMA2-7B to a sparse MoE by adding 8 LoRA expert adapters with a routing module. Trained on 194k examples (Self-RAG's 150k + 28k multi-hop). Achieves HotpotQA EM 63.3% vs. Self-RAG's 40.2%, MuSiQue EM 41.6% vs. 22.1%. Requires ~40 GPU-days on A100 80GB.

#### DACL-RAG (arXiv:2505.10493, 2025)

Data Augmentation with Curriculum Learning — starts training on oracle-only examples, progressively introduces distractors. More sample-efficient than RAFT for small labeled budgets.

---

### Practical Implementation for Gemma 4B on Small Compute

#### Can RAFT Be Applied to Gemma 4B?

Yes, with QLoRA. RAFT is fine-tuning-framework-agnostic.

| GPU | VRAM | Gemma 4B QLoRA | Training time (10k examples) |
|---|---|---|---|
| RTX 3090 / 4090 | 24 GB | Comfortable | ~2–4 hours |
| A10G | 24 GB | Comfortable | ~2–4 hours |
| T4 | 16 GB | Tight (use Unsloth) | ~6–12 hours |

Recommended LoRA config: r=16, α=32, target modules = q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj, down_proj. Unsloth reduces VRAM by ~60% and speeds training 2–5×.

**Important:** 4-bit NF4 quantization at inference degrades long-context performance by up to 59% (arXiv:2505.20276). If VRAM allows, deploy at 8-bit — 0.8% quality cost vs. up to 59% at 4-bit.

No peer-reviewed paper has applied RAFT or Self-RAG to Gemma 4B specifically. The MedGemma report (arXiv:2507.05201) applied standard medical SFT (no RAG-aware training) to Gemma 4B. Community RAFT implementations for Gemma-3-4B exist informally.

#### Constructing Synthetic Training Data from the Guideline Corpus

For a corpus with no existing labeled (question, retrieved passage, answer) triples:

1. **Question generation:** For each chunk, prompt GPT-4o-mini with few-shot nurse-style examples to generate 3–5 questions — one factual ("What is the recommended dose of misoprostol for PPH?"), one procedural ("What steps should be followed when...?"), one scenario-based ("A patient presents with X — what does the guideline recommend?").

2. **CoT answer generation:** Prompt GPT-4o with (question, oracle chunk) to generate a RAFT-format answer citing verbatim evidence using `##begin_quote## ... ##end_quote##` markers.

3. **Distractor retrieval:** Run the deployed retriever for each question; take top-5 results excluding the oracle chunk.

4. **Oracle/no-oracle mix:** 80% examples include one oracle + 4 distractors; 20% include only distractors (trains fallback behavior).

5. **Guideline conflict handling:** For the ~20–30 WHO vs. Zanzibar MOH divergence cases (misoprostol dosing, MgSO4, PMTCT, anaemia thresholds, malaria in pregnancy), create separate training examples per source with explicit attribution. This teaches the model to cite the source rather than silently pick one.

**Scale estimate:** MAMAI corpus ~10k chunks × 3 questions/chunk = ~30k examples. **Cost estimate: ~$30–50 with GPT-4o-mini.**

#### Existing Medical RAG Fine-Tuning Data

No dataset provides (question, retrieved passage from WHO/Tanzania/Zanzibar MOH guidelines, answer) triples. The closest available:

| Dataset | Size | Notes |
|---|---|---|
| MedQuAD | 47,457 QA pairs | No retrieved passages; usable for general medical SFT |
| MedRAG corpora | PubMed, StatPearls, textbooks | Retrieved passage + question + answer constructable via MedRAG toolkit |
| AfriMed-QA | ~25k QA | Closest to MAMAI population; no retrieved passages |
| MedMCQA | 194k MCQ | Indian medical entrance; limited guideline coverage |

Synthetic generation from the production corpus is the appropriate primary path.

---

### Measuring Context Utilization

End-task accuracy alone is insufficient — a model that ignores context can still score well on questions it already knows parametrically.

#### Decomposed ΔEM (Primary)

Run the model without retrieval first to classify each question as Known (correct) or Unknown (incorrect). Then measure:

> ΔEM_net = p_unk × ΔEM_unk + p_kn × ΔEM_kn

After RAFT fine-tuning, the target is: ΔEM_unk ↑ (more Unknown questions rescued by context) and ΔEM_kn ~0 or positive (Known questions not destroyed by added context).

#### Counterfactual Substitution Test

Replace the correct answer in the oracle passage with a plausible counterfactual (e.g., "2 units of oxytocin" → "5 units of oxytocin"). If the model follows the context, it outputs the counterfactual. Context Utilization Rate = fraction of cases where the model output tracks the counterfactual. Before RAFT: most 4B models will output the parametrically known value. After RAFT: they should follow the passage.

#### MiniCheck Faithfulness (Already in MAMAI Evaluation Plan)

The MiniCheck sentence-level support rate from §3.1 of `mamai-quality-evaluation.md` measures output faithfulness to retrieved context — the generator-side measure of context utilization. No additional tooling needed; run the same pipeline pre- and post-RAFT.

#### Probing Classifier on Hidden States (Mechanistic, Optional)

Kassner et al. (arXiv:2602.22787, 2026): a logistic regression probe trained on final-layer hidden states at the first generation token achieves Macro-F₁ = 0.96 in distinguishing parametric vs. contextual knowledge attribution, generalizing across tasks without fine-tuning. Use to directly measure whether RAFT shifted the model's internal knowledge channel from parametric-dominant to context-dominant.

---

### Key Papers for This Section

| Paper | Venue/Year | Finding |
|---|---|---|
| Pandey, arXiv:2603.11513 | 2026 | Sub-7B standard RAG: −3pp net; 85% of oracle retrieval wasted; irrelevant generation is the dominant failure |
| Shi et al., arXiv:2410.10796 | ICLR 2025 | Context-Parametric Inversion: instruction fine-tuning progressively suppresses context reliance |
| Liu et al., arXiv:2307.03172 | TACL 2024 | "Lost in the Middle": U-shaped attention; middle-context information systematically underweighted |
| ReDeEP, arXiv:2410.11414 | 2024 | FFN parametric dominance + Copying Head failure is the mechanistic cause of RAG hallucination |
| arXiv:2505.20276 | EMNLP 2025 | 4-bit quantization degrades long-context task performance up to 59%; 8-bit costs only ~0.8% |
| Asai et al., arXiv:2310.11511 | ICLR 2024 | Self-RAG: reflection tokens + adaptive retrieval; 7B beats ChatGPT on QA and fact verification |
| Zhang et al., arXiv:2403.10131 | 2024 | RAFT: distractor-robust SFT; 7B beats GPT-3.5+RAG on domain-specific tasks |
| Lin et al., arXiv:2310.01352 | ICLR 2024 | RA-DIT: joint LM + retriever fine-tuning on 1.2M instruction pairs |
| Open-RAG, arXiv:2410.01782 | EMNLP 2024 | MoE extension of Self-RAG; HotpotQA EM 63.3% vs. Self-RAG 40.2% |
| Tiny-Critic RAG, arXiv:2603.00846 | 2026 | 1.7B LoRA routing critic; F1=0.912; trainable on RTX 4090 in 42ms latency |
| Kassner et al., arXiv:2602.22787 | 2026 | Linear probing of hidden states distinguishes parametric vs. contextual attribution with F₁=0.96 |
| Finetune-RAG, arXiv:2505.10792 | 2025 | +21.2% from 1,653 examples in 20 steps; minimal RAFT works |
