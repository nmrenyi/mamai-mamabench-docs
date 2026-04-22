# MAMAI — Latency Evaluation

*System under evaluation: Gemma 4 E4B + RAG over WHO / Tanzania MOH / Zanzibar MOH guidelines, deployed as a medical-advice chatbot for nurses and midwives in Zanzibar*

*Latency is evaluated on the actual deployment hardware. Results obtained on a different machine are not meaningful for deployment decisions.*

---

## 1. What is being evaluated

The latency of the deployed RAG pipeline is evaluated component by component, following the same bottom-up structure as the quality evaluation: retriever first, generator in isolation, then the full end-to-end system. This separation identifies where latency bottlenecks sit — retrieval, generation, or both.

**Primary variable: input length** (query token count), across three buckets:
- Short: < 20 tokens
- Medium: 20–50 tokens
- Long: > 50 tokens

**Query set.** The mamaretrieval query set is used as the benchmark input — realistic nurse/midwife queries grounded in the deployment domain.

**Repeats.** Each query is run 30 times per condition. Discard the first 5 runs per session as warmup (to prime JIT and shader caches). 30 repeats is the minimum for stable P50/P90/P99 estimates.

**Metrics reported for every condition.** Latency is reported as percentiles across all runs, not as a mean. P50 is the median — the typical experience. P90 means 90% of queries finish at or below this time. P99 captures the worst-case a user realistically hits; a nurse submitting many queries per day will encounter P99 regularly. In addition to percentiles, plot the full latency distribution (histogram or CDF) for each condition — a bimodal or skewed distribution reveals structure (e.g. thermal throttling onset) that percentiles alone hide.

---

## 2. Retrieval latency

**What is measured.** Time from raw query string to ranked list of retrieved chunk IDs. Covers query embedding + vector search only.

**Variables.**
- Input length: short / medium / long buckets
- top-k ∈ {1, 3, 5, 10} — varying top-k changes the number of retrieved chunks and therefore the context length fed to the generator. Testing it here characterises the retrieval latency curve and informs the production top-k choice.

**Metrics.**
- P50 / P90 / P99 retrieval latency (ms) per input length × top-k
- Embedding throughput (tokens/sec)
- Latency distribution plot per condition

---

## 3. Generation latency

**What is measured.** Time from tokenised input to final generated token. Covers model inference only — retrieval is excluded. The input is query + context (fixed oracle chunks from mamaretrieval), so input length here means total token count fed to the model.

**Variables.**
- Input length in tokens (short / medium / long)

**Cold start vs. warm start.** Measured as two separate conditions:
- **Cold start**: first inference after model load, before any prior queries in the session. Model weights may not be fully paged into memory.
- **Warm start**: subsequent inferences in the same session, after warmup. This is the steady-state experience.

A nurse's first query after opening the app is a cold start; everything after is warm. Both matter.

**Metrics.**
- Time-to-first-token (TTFT) in ms — perceived responsiveness if the app streams output
- Total generation time (ms)
- Decode throughput (tokens/sec)
- P50 / P90 / P99 for each metric, separately for cold and warm start
- Latency distribution plot

---

## 4. End-to-end latency

**What is measured.** Wall-clock time from query submission to final token of response. Covers retrieval + generation as a single pipeline.

**Variables.**
- Input length: short / medium / long buckets
- RAG vs. no-RAG — isolates the retrieval overhead contribution to end-to-end latency

**Sustained load condition.** Run 30+ queries back-to-back with no cooldown between them. This simulates a nurse in a busy ward submitting queries in rapid succession and captures latency degradation under thermal throttling. Compare P50/P90/P99 from the sustained condition against the standard (cooled) condition — the gap quantifies throttling impact on real-world use.

**Metrics.**
- P50 / P90 / P99 end-to-end latency (ms) per input length bucket
- Retrieval share vs. generation share (%) — shows which component dominates
- Latency distribution plot: standard vs. sustained load overlaid

---

## 5. Hardware and environment

Record the following before running any measurement. These are fixed for a given evaluation run and must be reported alongside all results.

| Spec | Value |
|---|---|
| Device model | |
| SoC / chipset | |
| CPU model | |
| CPU base / boost frequency | |
| CPU core count (big / little) | |
| RAM (total) | |
| GPU model | |
| Android version | |
| Inference framework (e.g. LiteRT-LM) | |
| Model quantization (e.g. INT4, INT8, FP16) | |
| RAG retrieval top-k (production value) | |

**CPU temperature.** Record idle temperature before the run and peak temperature during the run via ADB:

```
adb shell cat /sys/class/thermal/thermal_zone*/temp
```

If peak temperature triggers throttling — detectable as a step-change in throughput mid-run — flag it explicitly. Throttled numbers are not representative of steady-state deployment.

**Memory footprint.** Record heap memory at baseline (model loaded, no inference) and peak (during active inference). Flag if peak memory causes swapping.

**Multiple devices.** Run on at least two devices: the primary benchmark device and a lower-end device representative of actual Zanzibar deployment hardware. Report all results per device separately.

---

## 6. Reporting template

For each component and condition:

| Input length | P50 (ms) | P90 (ms) | P99 (ms) |
|---|---|---|---|
| Short (< 20 tok) | | | |
| Medium (20–50 tok) | | | |
| Long (> 50 tok) | | | |

Accompany each table with a latency distribution plot. For end-to-end, overlay standard vs. sustained load distributions. Report hardware spec table, CPU temperature (idle / peak), memory (baseline / peak), and any throttling or swap events.

---

## 7. Battery and thermal impact

This section evaluates two deployment-critical concerns specific to the Zanzibar context: how much battery a session of queries consumes, and how the device thermals evolve over sustained use. A device used in a warm ward without reliable charging will face conditions significantly worse than a lab benchmark.

### 7.1 Battery consumption

**Method.** Run the benchmark unplugged. Record battery level via ADB before and after a fixed session (e.g. 30 end-to-end queries):

```
adb shell dumpsys battery
```

For energy consumption over the session window:

```
adb shell dumpsys batterystats
```

**Metrics.**
- Battery drain (%) per session
- Estimated queries per full charge cycle

**Caveat.** Battery measurement is noisy on a plugged-in device — the benchmark must run unplugged for meaningful results. Long runs may require session splitting to avoid full discharge.

### 7.2 Thermal profile over time

**Method.** Log CPU temperature continuously throughout the benchmark run — sample every 10 seconds via ADB:

```
adb shell cat /sys/class/thermal/thermal_zone*/temp
```

Additionally, query the Android thermal throttling status at each sample point:

```
adb shell dumpsys thermalservice
```

This reports throttling severity levels directly (NONE / LIGHT / MODERATE / SEVERE / CRITICAL), not just temperature.

**Metrics.**
- Temperature over time plot (°C vs. elapsed time)
- Throttling severity level over time, overlaid on the latency-over-time plot
- Time to first throttling onset (seconds from session start)
- Temperature at throttling onset
- Recovery time after cooldown resumes

**Why this matters.** The existing benchmark shows 5–15% latency degradation over 108 runs in lab conditions. In a 35°C ward without air conditioning, throttling onset will be earlier and more severe. The thermal profile over time gives a deployment-realistic picture of how the system behaves across a full shift, not just a short benchmark run.
