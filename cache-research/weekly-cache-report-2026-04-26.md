# Weekly Cache Research Report — 2026-04-26

**Run type:** Normal weekly run. Prior report: [`weekly-cache-report-2026-04-21.md`](./weekly-cache-report-2026-04-21.md). Search horizon: 2026-04-20 → 2026-04-26 (past 7 days, ICLR 2026 week). Top 5 entries selected.

**Scope:** distributed caching, KV cache, caching for inference, storage-system caching.

**Selection criteria:** novelty of mechanism, potential systems/production impact, and whether the work measured runtime efficiency on a realistic workload — not only model-quality metrics.

**Organization:** (A) production measurements / empirical studies — real deployments, real traces, real hardware — and (B) academic / idea-forward work — novel mechanisms typically evaluated on benchmarks or simulated traces.

---

## A. Production measurements and empirical studies

### 1. KServe × llm-d × vLLM: prefix-cache-aware routing measured on Llama 3.1 70B / MI300X (Apr 21, 2026)

- Reference: [llm-d blog — Production-Grade LLM Inference at Scale with KServe, llm-d, and vLLM](https://llm-d.ai/blog/production-grade-llm-inference-at-scale-kserve-llm-d-vllm). Companion: [Red Hat Developer write-up (Apr 21, 2026)](https://developers.redhat.com/articles/2026/04/21/kserve-llm-d-optimized-gen-ai-inference).
- Summary: Joint Red Hat / Tesla post on running multi-tenant LLM serving via KServe's `LLMInferenceService` / `LLMInferenceConfig` CRDs backed by llm-d's Envoy + Gateway-API Inference Extension. The prefix-cache-aware router steers requests to the vLLM replica that already holds a matching prefix, so the second-level KV-cache layer (LMCache) actually gets used instead of being duplicated across pods.
- Novelty: Low as mechanism (prefix-aware routing has shipped since 2025), but high as the first end-to-end Kubernetes-native, vendor-neutral reference for it. Notable that this is now being rolled into the standard Gateway API rather than a custom CRD.
- Impact: High for operators. Establishes prefix-cache-aware routing as the default rather than an opt-in optimisation, and gives Tesla / Red Hat a public production case-study to point at.
- Runtime evaluation: Yes, on a real fleet — Llama 3.1 70B on 4× AMD MI300X with `tensor-parallel-size=4`, `gpu-memory-utilization=0.90`, `max-model-len=65536`. Reports **3× output tok/s and 2× TTFT reduction** after enabling prefix-aware routing vs. round-robin. No fairness or tail-latency numbers; measurement is throughput-and-mean focused.

### 2. SAW-INT4: System-Aware 4-Bit KV-Cache Quantization for Real-World LLM Serving (Apr 21, 2026)

- Reference: Jia, Li et al., [arXiv:2604.19157](https://arxiv.org/abs/2604.19157).
- Summary: Audits the 2024–2026 KV-quantization literature against practical serving constraints (paged KV layout, regular memory access, fused attention) and shows that most "best-on-paper" methods (vector quantization, Hessian-aware quantization) lose almost all of their advantage once those constraints are honoured. The viable design that survives is a simple token-wise INT4 with **block-diagonal Hadamard rotation**, plus a fused rotation-quantize kernel that drops into the paged KV layout with zero measurable end-to-end overhead.
- Novelty: Medium-high. The mechanism is restrained on purpose — the contribution is the system-level audit and the surprising "complex methods don't pay" result.
- Impact: High for inference engines (vLLM, SGLang, TensorRT-LLM) currently choosing between INT4 schemes. Reframes 4-bit KV as a kernel-engineering problem rather than a quantizer-search problem.
- Runtime evaluation: Yes — accuracy across multiple models/benchmarks plus throughput at realistic concurrency vs. plain INT4 baseline. Claims near-lossless accuracy at INT4 throughput; absolute serving numbers (TTFT, TBT, p99) are lighter than e.g. Mooncake-style reports.

---

## B. Academic / idea-forward work

### 3. TurboQuant: Online Vector Quantization with Near-Optimal Distortion Rate (ICLR 2026, presented Apr 25, 2026)

- Reference: Zandieh, Daliri, Hadian, Mirrokni — [OpenReview tO3ASKZlok](https://openreview.net/forum?id=tO3ASKZlok), [arXiv:2504.19874](https://arxiv.org/abs/2504.19874), [ICLR 2026 poster page](https://iclr.cc/virtual/2026/poster/10006985), [Google Research blog](https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/).
- Summary: A data-oblivious, online vector quantizer that hits near-optimal distortion at any bit-width by (1) randomly rotating inputs to induce a Beta-distributed coordinate distribution, (2) applying optimal per-coordinate scalar quantizers, and (3) correcting inner-product bias with a 1-bit QJL residual. Applied to KV cache, it yields ~3 bits/value with negligible attention-fidelity loss.
- Novelty: High. Brings a sharp information-theoretic result (near the rate-distortion bound, factor ≈2.7) to a problem the systems community has been attacking heuristically. Same primitive also targets vector search.
- Impact: Potentially very high. Already multiple independent reproductions (Triton kernels, vLLM integration patches) within the ICLR week. If the H100 numbers (≥6× memory cut, up to 8× faster attention-logit compute) hold under realistic batched serving, this displaces a wide swath of 2024–2025 KV-quant baselines.
- Runtime evaluation: Partial. The paper itself is theory-led with attention-fidelity and microbenchmark results on H100; the open-source ports add Triton kernel timings and ad-hoc vLLM throughput numbers, but a full TTFT/TBT/SLO study under continuous batching is still missing — the obvious next paper to write.

### 4. SparKV: Overhead-Aware KV Cache Loading for Efficient On-Device LLM Inference (Apr 23, 2026)

- Reference: Liu et al., [arXiv:2604.21231](https://arxiv.org/abs/2604.21231).
- Summary: For on-device inference with a cloud assist, SparKV decides per chunk whether to **stream** the KV from cloud or **recompute** it locally, then overlaps the two paths. A KV Chunk Scheduler does dependency-aware planning, a small MLP overhead model predicts per-chunk compute latency from attention-sparsity features, and a runtime controller migrates chunks between paths as wireless bandwidth and edge headroom fluctuate. Functionally, this is CacheGen / "compute-or-load" generalised to a flaky-network mobile setting.
- Novelty: Medium-high. The mechanism is incremental, but the framing — KV transport as a two-path scheduling problem with online refinement — is cleaner than prior compute-or-load work and explicitly treats wireless variance as a first-class signal.
- Impact: High for on-device / edge LLM workloads where TTFT and energy both matter and connectivity is unstable; less directly relevant to datacenter serving.
- Runtime evaluation: Yes — across multiple LLMs, datasets, and edge devices: **1.3–5.1× TTFT reduction** and **1.5–3.3× per-request energy reduction** with negligible quality loss vs. full-load and full-recompute baselines.

### 5. How Much Cache Does Reasoning Need? Depth–Cache Tradeoffs in KV-Compressed Transformers (Apr 20, 2026)

- Reference: Wang, [arXiv:2604.17935](https://arxiv.org/abs/2604.17935).
- Summary: A theory paper that asks the dual of "how small can the cache get without losing accuracy?" — namely, *given* a compressed cache of size `s`, attention dimension `m`, `H` heads, `p`-bit precision, and a locality-respecting cache controller (which captures essentially all 2024–2026 KV-compression schemes), how much network depth is needed to solve `k`-hop pointer chasing on `n` tokens? The author proves a product-form depth lower bound and a matching upper bound via windowed pointer doubling.
- Novelty: High. Provides the first formal depth-vs-cache lower bound that explicitly covers the locality-respecting compression class (eviction, quantization, transform-coded KV alike). Closes the "are we compressing too aggressively?" question for a non-trivial reasoning primitive.
- Impact: Medium-high near term, foundational long-term. Direct prescriptive value for KV-compression designers: aggressive compression imposes a depth tax on multi-hop reasoning that no clever controller can erase.
- Runtime evaluation: None (theory paper). Worth reading alongside the empirical "Hold Onto That Thought" reasoning-quality benchmarks ([arXiv:2512.12008](https://arxiv.org/abs/2512.12008)) for the corresponding measurement side.

---

## Cross-cutting observations (this week)

- **ICLR week skewed the field toward compression.** TurboQuant (theory-grounded VQ), SAW-INT4 (system-aware INT4), and Wang's depth–cache lower bound landed in the same seven days. The conversation has shifted from "which quantizer wins on LongBench?" to "which quantizer survives a paged-attention kernel?" and "what does the cache size *fundamentally* cost?"
- **Production stacks are converging on prefix-cache-aware routing as table stakes.** The KServe + llm-d + vLLM post is the third independent claim of "~2–3× output tok/s, ~2× TTFT" from prefix-aware routing in 2025–2026 (after the original llm-d blog and the LMCache × Dynamo report). This is no longer a research direction; it's an operational default.
- **Edge / on-device KV is a real subfield now.** SparKV explicitly treats wireless variance as a first-class scheduling input, joining KVSwap and on-device 4-bit quantization (e.g. "Don't Waste Bits!" — [arXiv:2604.04722](https://arxiv.org/abs/2604.04722)) in shaping a distinct edge-inference cache literature with its own constraints (energy, bandwidth, intermittent connectivity).
- **Runtime-evaluation gap persists for theory-led work.** TurboQuant's ICLR paper measures attention fidelity and microbench throughput; a full continuous-batching TTFT/TBT/SLO study is still owed — this is the clearest "next paper to write" of the week.

## References

- [llm-d — Production-Grade LLM Inference at Scale with KServe, llm-d, and vLLM (Apr 21, 2026)](https://llm-d.ai/blog/production-grade-llm-inference-at-scale-kserve-llm-d-vllm)
- [Red Hat Developer — Combining KServe and llm-d (Apr 21, 2026)](https://developers.redhat.com/articles/2026/04/21/kserve-llm-d-optimized-gen-ai-inference)
- [arXiv:2604.19157 — SAW-INT4 (Apr 21, 2026)](https://arxiv.org/abs/2604.19157)
- [OpenReview tO3ASKZlok — TurboQuant (ICLR 2026)](https://openreview.net/forum?id=tO3ASKZlok) · [arXiv:2504.19874](https://arxiv.org/abs/2504.19874) · [ICLR 2026 poster page](https://iclr.cc/virtual/2026/poster/10006985) · [Google Research blog](https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/)
- [arXiv:2604.21231 — SparKV (Apr 23, 2026)](https://arxiv.org/abs/2604.21231)
- [arXiv:2604.17935 — Depth–Cache Tradeoffs (Apr 20, 2026)](https://arxiv.org/abs/2604.17935)

### Additional context (not selected for top 5, but noted this week)

- [arXiv:2604.15039 — Prefill-as-a-Service / PrfaaS (Apr 16, 2026)](https://arxiv.org/abs/2604.15039) — cross-datacenter KV transfer for PD disaggregation (Moonshot AI / Tsinghua); just outside the 7-day window but closely related to the production-routing entry above.
- [arXiv:2604.10539 — IceCache (Apr 14, 2026)](https://arxiv.org/abs/2604.10539) — semantic-cluster-aware paged KV management; 99% accuracy at 25% token budget on LongBench.
- [arXiv:2604.08426 — KV Cache Offloading for Context-Intensive Tasks (Apr 11, 2026)](https://arxiv.org/abs/2604.08426) — Text2JSON benchmark exposing offloading regressions on extraction-heavy workloads; useful counterweight to SparKV-style claims.
- [arXiv:2604.19769 — TTKV (temporal-tiered KV)](https://arxiv.org/abs/2604.19769) and [arXiv:2604.18529 — HybridGen (CPU–GPU hybrid)](https://arxiv.org/abs/2604.18529) — both relevant tiered-KV neighbours, lower priority for this week.
- [arXiv:2512.12008 — Hold Onto That Thought](https://arxiv.org/abs/2512.12008) — empirical companion to entry #5 on reasoning-under-compression.
