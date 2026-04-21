# Weekly Cache Research Report — 2026-04-21

**Run type:** Catch-up (first run — no prior reports found in `cache-research/`). Search horizon expanded to the past ~12 months; top 20 entries selected.

**Scope:** distributed caching, KV cache, caching for inference, storage-system caching.

**Selection criteria:** novelty of mechanism, potential systems/production impact, and whether the work actually measured runtime efficiency (latency, throughput, memory, I/O, SLO) on a realistic workload rather than only model-quality metrics.

**Organization:** entries are split into (A) production measurements and empirical studies — real deployments, real traces, real hardware — and (B) academic / idea-forward work — novel mechanisms typically evaluated on benchmarks or simulated traces.

---

## A. Production measurements and empirical studies

### 1. Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving (FAST '25 Best Paper)

- Reference: Qin et al., FAST '25 / arXiv:2407.00079 — [arxiv](https://arxiv.org/abs/2407.00079), [USENIX FAST '25](https://www.usenix.org/conference/fast25/presentation/qin), [code](https://github.com/kvcache-ai/Mooncake).
- Summary: Disaggregates prefill and decode clusters and pools CPU DRAM, SSD, and the RDMA fabric across the GPU cluster into a shared "KVCache pool." An early-rejection scheduler protects SLOs under overload.
- Novelty: High. Reframes LLM serving as a storage system problem ("trade storage for compute") and publishes a production trace from Kimi. Open-sourced Transfer Engine (Nov 2024) and Mooncake Store (Mar 2025).
- Impact: Very high. Adopted or influenced vLLM, SGLang, and llm-d; cited as the reference disaggregated design. Reports up to 525% throughput increase in simulation of real traces while meeting SLOs, and ~75% more served requests under real workloads.
- Runtime evaluation: Yes — TTFT, TBT, end-to-end throughput, and SLO attainment on Kimi production traces, on H800-class fleets, with real RDMA interconnect and staged DRAM/SSD. Mooncake Store reports up to 2.36× cache hit rate vs. local cache and ~48% prefill time savings.

### 2. Mooncake in production: Kimi K2 on 128×H200 (2025 operational update)

- Reference: Mooncake blog and repo updates, Jul 2025 — [Mooncake site](https://kvcache-ai.github.io/Mooncake/).
- Summary: Operational write-up of the same stack scaled to Kimi K2 (1T parameters). PD disaggregation + large-scale expert parallelism on 128×H200 reportedly delivers 224k tok/s prefill and 288k tok/s decode throughput. The Mooncake P2P Store refreshes the 1T-parameter model across thousands of GPUs in ~20 s.
- Novelty: Low-medium as a system artifact (same architecture), high as an empirical data point at K2 scale.
- Impact: High signal for practitioners — concrete numbers from a frontier-scale deployment.
- Runtime evaluation: Yes — production throughput and parameter-refresh latency; not a controlled benchmark.

### 3. LMCache: An Efficient KV Cache Layer for Enterprise-Scale LLM Inference

- Reference: Liu et al., arXiv:2510.09665, Oct 2025 — [arxiv](https://arxiv.org/abs/2510.09665), [tech report](https://lmcache.ai/tech_report.pdf), [code](https://github.com/LMCache/LMCache).
- Summary: Cross-engine KV cache layer sitting outside the GPU, supporting prefix and non-prefix reuse across vLLM/SGLang instances with batched data movement and compute-I/O pipelining behind a modular connector (CPU, NVMe, S3, Ceph, remote).
- Novelty: Medium-high. Generalizes "prefix caching" to an engine-agnostic tiered cache (GPU → CPU → SSD → remote) with a production API and connectors.
- Impact: High. De-facto open-source KV-cache middleware for enterprise inference; referenced by llm-d and NVIDIA Dynamo as their cache layer.
- Runtime evaluation: Yes — TTFT, throughput, and prefix hit ratio under enterprise traces; reports ~10× speedup vs. baseline vLLM on prefix-heavy workloads in the production stack.

### 4. HCache: Fast State Restoration in LLM Serving (EuroSys '25)

- Reference: Gao et al., arXiv:2410.05004 — [arxiv](https://arxiv.org/abs/2410.05004), [EuroSys '25](https://dl.acm.org/doi/10.1145/3689031.3696072).
- Summary: Restores LLM state from intermediate activations (rather than raw KV or retokenized prefix) with a bubble-free restoration scheduler that balances compute and I/O and a chunk-based storage manager for layout mismatch.
- Novelty: Medium-high. Treats cache restoration as a resource-scheduling problem, not just a data-movement one.
- Impact: High for any tiered-cache design with eviction — directly relevant to LMCache/Mooncake-class systems.
- Runtime evaluation: Yes — up to 1.93× TTFT reduction vs. KV offload at 1.92–2.40× less storage; up to 5.73× TTFT reduction vs. token recomputation on realistic multi-round/RAG workloads.

### 5. Towards Efficient Flash Caches with Emerging NVMe Flexible Data Placement SSDs (EuroSys '25)

- Reference: arXiv:2503.11665 — [arxiv](https://arxiv.org/abs/2503.11665), [ACM](https://dl.acm.org/doi/10.1145/3689031.3696091).
- Summary: Extends CacheLib, Meta's production flash cache, to use NVMe FDP placement hints so that data with similar lifetimes is colocated on SSD, drastically cutting device write amplification.
- Novelty: Medium-high. First published CacheLib × FDP integration with a production-aligned evaluation.
- Impact: High. Reports 2× SSD-cost reduction and 4× embodied-carbon reduction at near-zero hit-rate cost — directly actionable for large-scale cache operators.
- Runtime evaluation: Yes — replays production traces from Meta and Twitter; measures WAF, hit rate, throughput, and power.

### 6. NVIDIA Dynamo + partner stacks: measured KV-offload wins in production

- Reference: NVIDIA Dynamo announcement (GTC 2025) — [NVIDIA blog](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/), [VAST × CoreWeave writeup](https://www.vastdata.com/blog/nvidia-dynamo-vast-scalable-optimized-inference), [LMCache × Dynamo](https://blog.lmcache.ai/2025-09-18-dynamo-lmcache/).
- Summary: Dynamo is a distributed inference framework with KV-aware routing and hierarchical KV offload (HBM → DRAM → local SSD → networked storage). Integrates with LMCache as its cache layer.
- Novelty: Low as mechanism (aggregates known techniques); high as a reference production stack.
- Impact: High — VAST + CoreWeave report GPU efficiency +90% and TTFT up to 20× better when offloaded KV is served from network storage rather than recomputed. Now part of NVIDIA's AI Enterprise path.
- Runtime evaluation: Yes, but vendor-published — full numbers require access to vendor reports; the methodology is production workload replay.

### 7. llm-d: prefix caching measured at cluster scale (blog, 2025)

- Reference: [llm-d blog — KV-Cache Wins You Can See](https://llm-d.ai/blog/kvcache-wins-you-can-see).
- Summary: Open-source, K8s-native LLM serving platform combining vLLM, LMCache, and a cache-aware router. The post reports hit-rate and TTFT improvements as traffic is routed to replicas holding the matching prefix.
- Novelty: Low-medium (integration); high as a concrete, reproducible open-stack production measurement.
- Impact: Medium-high; llm-d is a growing reference for multi-tenant serving with prefix-aware routing.
- Runtime evaluation: Yes — reports cache hit rate, TTFT, and throughput on realistic trace replays at cluster scale.

### 8. Prompt / context caching in production APIs (Anthropic, OpenAI, Google)

- References: [Anthropic prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching), [OpenAI prompt caching](https://platform.openai.com/docs/guides/prompt-caching), [Vertex AI context caching](https://cloud.google.com/vertex-ai/generative-ai/docs/context-cache/context-cache-overview), [Google implicit caching launch](https://mlq.ai/news/google-launches-implicit-caching-to-slash-ai-model-costs-for-developers/).
- Summary: Two diverging commercial models for request-level KV reuse. OpenAI and Gemini 2.5+ ship *implicit* caching (automatic, free discount on matched prefixes, ≥1024 tokens, 5–10 min–24 h TTL). Anthropic ships *explicit* caching controlled by `cache_control` (5-min TTL, refreshes on use).
- Novelty: Low as mechanism (prefix caching), high as public productization of KV reuse with published pricing.
- Impact: Very high economically — publicly stated 50–90% cost cuts and 80–85% latency cuts on cached prefixes; 10× price delta for cached vs. uncached tokens at Gemini 2.5.
- Runtime evaluation: Yes, per-provider, but the numbers are marketing-grade; `cachedContentTokenCount` (Gemini) and `cache_creation_input_tokens` / `cache_read_input_tokens` (Anthropic) give users first-party instrumentation.

### 9. RAGPulse: An Open-Source RAG Workload Trace (empirical)

- Reference: Wang et al., arXiv:2511.12979 — [arxiv](https://arxiv.org/abs/2511.12979).
- Summary: A real-world trace of a Q&A RAG system revealing temporal locality and hot-document patterns; intended as a benchmark for cache, batching, and routing research.
- Novelty: Medium — the trace is the contribution, but fills a real gap (RAG has lacked a Mooncake-style public trace).
- Impact: High as an enabler; future cache/scheduler papers now have a shared RAG-serving reference workload.
- Runtime evaluation: N/A (trace paper); provides the substrate to evaluate other systems.

### 10. Optimizing SSD Caches for Cloud Block Storage Using ML (production traces)

- Reference: Wang et al., arXiv:2501.14770 — [arxiv](https://arxiv.org/abs/2501.14770); companion work: CMU Baleen thesis [CMU-CS-24-152](http://reports-archive.adm.cs.cmu.edu/anon/2024/CMU-CS-24-152.pdf).
- Summary: ML-driven admission/write-skip for SSD caches in cloud block storage; predicts cold writes to skip the flash layer, improving hit rate and endurance.
- Novelty: Medium — builds on a rich ML-admission literature; contribution is the production-trace evaluation and write-skip policy.
- Impact: High for cloud storage operators (endurance is a real cost driver). Outside the LLM bubble but directly relevant to "caching" as a systems problem and a useful counterpoint to KV-cache work.
- Runtime evaluation: Yes — hit rate, write amplification, and SSD endurance on real block-storage traces.

---

## B. Academic / idea-forward work

### 11. CacheBlend: Fast LLM Serving for RAG with Cached Knowledge Fusion (EuroSys '25)

- Reference: Yao et al., arXiv:2405.16444 — [arxiv](https://arxiv.org/abs/2405.16444), [EuroSys '25](https://dl.acm.org/doi/10.1145/3689031.3696098).
- Summary: Reuses precomputed KV for arbitrary (non-prefix) text chunks in RAG and selectively recomputes a small subset of tokens to repair cross-chunk attention. The "selective recompute" idea lets RAG escape the prefix-only limitation of vanilla prefix caching.
- Novelty: High. Opens the door to cross-query KV reuse in RAG while preserving output parity.
- Impact: High. Cited widely; informs Cache-Craft, RAGBoost, and production chunk-caching designs.
- Runtime evaluation: Yes — 2.2–3.3× TTFT reduction and 2.8–5× throughput improvement vs. full KV recompute with no quality loss.

### 12. CacheGen: KV Cache Compression and Streaming for Fast LLM Serving (SIGCOMM '24, updated deployments '25)

- Reference: Liu et al., arXiv:2310.07240 — [arxiv](https://arxiv.org/abs/2310.07240), [SIGCOMM '24](https://dl.acm.org/doi/10.1145/3651890.3672274), [LMCache deployment blog](https://blog.lmcache.ai/en/2025/07/31/cachegen-store-your-kv-cache-on-disk-or-s3-load-blazingly-fast/).
- Summary: Custom tensor encoder that exploits KV's statistical structure to produce a bitstream representation with adaptive per-bandwidth levels, for shipping KVs from disk or over the network.
- Novelty: High. Treats KV transport as a streaming / adaptive-bitrate problem.
- Impact: High — the canonical reference for "store and ship KV like a compressed asset"; now implemented inside LMCache with S3/disk backends.
- Runtime evaluation: Yes — 3.5–4.3× KV size reduction and 3.2–3.7× total fetch+process delay reduction with negligible quality loss.

### 13. Cache-Craft: Managing Chunk-Caches for Efficient RAG

- Reference: Agarwal et al., arXiv:2502.15734 — [arxiv](https://arxiv.org/abs/2502.15734).
- Summary: Reuses precomputed per-chunk KV across queries and orderings in RAG, handling positional encoding and cross-chunk attention safely enough to ship.
- Novelty: High. Extends CacheBlend's recipe into a deployable chunk-caching layer.
- Impact: High for RAG-heavy production workloads where the retrieved chunk set is often overlapping.
- Runtime evaluation: Yes — TTFT, throughput, and quality parity vs. vLLM on LLaMA-3-8B/70B with continuous batching.

### 14. KV Cache Transform Coding for Compact Storage (KVTC)

- Reference: Staniszewski & Łańcucki, arXiv:2511.01815, Nov 2025 — [arxiv](https://arxiv.org/abs/2511.01815).
- Summary: PCA decorrelation + adaptive quantization + entropy coding for K/V tensors, targeting *stored* KV caches, not online attention.
- Novelty: Medium-high. Revives classical image-codec tools for KV; orthogonal to eviction/selection.
- Impact: High for tiered KV (LMCache, Mooncake Store, Dynamo) where cache size and bandwidth dominate cost; reports up to 20× compression at small quality loss.
- Runtime evaluation: Partial — strong accuracy-vs-ratio curves; end-to-end serving-latency measurements are lighter.

### 15. HACK: Homomorphic Acceleration via KV Compression for Disaggregated LLM Inference (SIGCOMM '25)

- Reference: [ACM SIGCOMM 2025](https://dl.acm.org/doi/10.1145/3718958.3750481).
- Summary: KV is compressed once and then attention operates *directly on the compressed form* ("homomorphic" in the sense of decompress-free compute), addressing the KV-transfer bottleneck in prefill/decode disaggregation.
- Novelty: High. Pushes compression past the offload/network tier into the compute kernel itself.
- Impact: Medium-high. JCT reduction up to 70.9% vs. disaggregated baseline; technique is compelling even if the HW/kernel requirements are nontrivial.
- Runtime evaluation: Yes — JCT, TTFT, and bandwidth utilization under disaggregated serving simulation.

### 16. Locality-aware Fair Scheduling in LLM Serving (DLPM / D²LPM)

- Reference: Cao et al., arXiv:2501.14312 — [arxiv](https://arxiv.org/abs/2501.14312).
- Summary: Deficit Longest Prefix Match — a deficit-round-robin-style scheduler that trades prefix-cache locality for per-tenant fairness without sacrificing throughput.
- Novelty: Medium-high. Cleanly models a three-way objective (fairness, locality, balance) using a Virtual Token Counter + LPM.
- Impact: High for multi-tenant serving where one tenant's hot prefixes can starve others.
- Runtime evaluation: Yes — throughput, p99 latency, and a fairness index across tenants on synthetic and real traces.

### 17. Ada-KV: Adaptive Budget Allocation for KV Cache Eviction

- Reference: Feng et al., arXiv:2407.11550 (updated 2025) — [arxiv](https://arxiv.org/abs/2407.11550).
- Summary: Allocates per-head eviction budgets from attention-pattern statistics rather than using a uniform budget across heads.
- Novelty: Medium. A clean empirical observation (heads differ) turned into a principled allocation rule.
- Impact: High — widely cited baseline now integrated into several KV-compression pipelines and comparison tables.
- Runtime evaluation: Yes — memory vs. quality across LongBench; throughput numbers are lighter but present.

### 18. SCBench: A KV Cache-Centric Analysis of Long-Context Methods

- Reference: Li et al., arXiv:2412.10319 — [arxiv](https://arxiv.org/abs/2412.10319).
- Summary: Benchmark that evaluates long-context methods along the four KV lifecycle stages — generation, compression, retrieval, loading — instead of collapsing everything into end-task accuracy.
- Novelty: High. Exposes that many long-context methods win on one stage and lose on another; the "layer-level sparsity" finding is widely referenced.
- Impact: High as a measurement framework for KV research; becoming a standard reference.
- Runtime evaluation: Yes — this *is* the runtime evaluation framework; measures attention shift and stage-wise cost.

### 19. InstInfer: In-Storage Attention Offloading on Computational Storage Drives

- Reference: Pan et al., arXiv:2409.04992 — [arxiv](https://arxiv.org/abs/2409.04992).
- Summary: Runs attention compute *inside* Computational Storage Drives, exploiting internal flash bandwidth and P2P transfers to bypass host PCIe.
- Novelty: High. Crosses the hardware boundary — attention inside the SSD controller — one of the few papers genuinely on the storage-system-caching frontier.
- Impact: Medium today, potentially high if CSDs become commodity; most relevant for cost-constrained long-context inference.
- Runtime evaluation: Yes — decoding throughput vs. host-offloaded baselines and PCIe saturation analysis.

### 20. Infinite-LLM / DistAttention: Distributed KVCache for Long Context

- Reference: Lin et al., arXiv:2401.02669 — [arxiv](https://arxiv.org/abs/2401.02669).
- Summary: DistAttention partitions attention across nodes and pools KV across CPU/GPU in a cloud fleet, aimed at elastic context growth.
- Novelty: Medium-high at publication; foundational for the subsequent Mooncake / LMCache wave.
- Impact: High as a predecessor architecture for the disaggregated-KV trend; still a useful comparison baseline.
- Runtime evaluation: Yes — end-to-end throughput and context-length reachability in a data-center setting.

---

## Cross-cutting observations

- **KV cache is the new storage tier.** Section A (Mooncake, LMCache, HCache, Dynamo, llm-d) and Section B's Infinite-LLM / HACK / KVTC all treat KV as a first-class distributed-storage problem — pooled DRAM/SSD over RDMA, hash-ring placement, SLO-aware scheduling, transform coding. This is the single clearest trend of the past year.
- **Production APIs have normalized prefix reuse.** OpenAI, Anthropic, and Gemini now publish cache-hit pricing and metadata; the "turn KV reuse into a user-visible 10× discount" move makes prefix caching a baseline expectation rather than an optimization.
- **Compression is shifting from eviction to coding.** KVTC (transform coding), CacheGen (streaming), HACK (compute-on-compressed), and Ada-KV (head-aware budgets) collectively move the field beyond the 2024 wave of eviction heuristics.
- **RAG caching is maturing as a subfield.** Cache-Craft + CacheBlend + RAGPulse (trace) indicate RAG-specific cache reuse (not just prefix) now has its own mechanisms and its own reference workload.
- **Runtime-efficiency reporting is uneven.** Systems-venue papers (Mooncake, LMCache, HCache, CacheLib-FDP, Cache-Craft) report TTFT/TBT/SLO on realistic traces. ML-venue KV-compression papers (KVTC, Ada-KV, EvolKV) still skew toward quality-vs-memory and often omit serving-latency numbers — a gap the field should close.
- **Storage-system-caching research continues in parallel** (ML for SSD admission in cloud block storage; CacheLib-FDP) and is starting to re-enter the LLM conversation via shared building blocks (CacheLib under LMCache; FDP-aware tiers under Dynamo). Cross-pollination is still limited but accelerating.

## References

- [arXiv:2407.00079 — Mooncake](https://arxiv.org/abs/2407.00079) · [USENIX FAST '25 page](https://www.usenix.org/conference/fast25/presentation/qin) · [github.com/kvcache-ai/Mooncake](https://github.com/kvcache-ai/Mooncake)
- [Mooncake project site / K2 update](https://kvcache-ai.github.io/Mooncake/)
- [arXiv:2510.09665 — LMCache](https://arxiv.org/abs/2510.09665) · [LMCache tech report](https://lmcache.ai/tech_report.pdf) · [github.com/LMCache/LMCache](https://github.com/LMCache/LMCache)
- [arXiv:2410.05004 — HCache (EuroSys '25)](https://arxiv.org/abs/2410.05004) · [ACM DL](https://dl.acm.org/doi/10.1145/3689031.3696072)
- [arXiv:2503.11665 — CacheLib + NVMe FDP (EuroSys '25)](https://arxiv.org/abs/2503.11665) · [ACM DL](https://dl.acm.org/doi/10.1145/3689031.3696091)
- [NVIDIA Dynamo announcement](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/) · [VAST × CoreWeave](https://www.vastdata.com/blog/nvidia-dynamo-vast-scalable-optimized-inference) · [LMCache × Dynamo](https://blog.lmcache.ai/2025-09-18-dynamo-lmcache/)
- [llm-d — KV-Cache Wins You Can See](https://llm-d.ai/blog/kvcache-wins-you-can-see) · [docs.vllm.ai prefix caching](https://docs.vllm.ai/en/stable/design/prefix_caching/)
- [Anthropic prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) · [OpenAI prompt caching](https://platform.openai.com/docs/guides/prompt-caching) · [Vertex AI context caching](https://cloud.google.com/vertex-ai/generative-ai/docs/context-cache/context-cache-overview) · [Gemini implicit caching](https://mlq.ai/news/google-launches-implicit-caching-to-slash-ai-model-costs-for-developers/)
- [arXiv:2511.12979 — RAGPulse](https://arxiv.org/abs/2511.12979)
- [arXiv:2501.14770 — ML for SSD caches in cloud block storage](https://arxiv.org/abs/2501.14770) · [CMU-CS-24-152 — Baleen thesis](http://reports-archive.adm.cs.cmu.edu/anon/2024/CMU-CS-24-152.pdf)
- [arXiv:2405.16444 — CacheBlend (EuroSys '25)](https://arxiv.org/abs/2405.16444) · [ACM DL](https://dl.acm.org/doi/10.1145/3689031.3696098)
- [arXiv:2310.07240 — CacheGen (SIGCOMM '24)](https://arxiv.org/abs/2310.07240) · [ACM DL](https://dl.acm.org/doi/10.1145/3651890.3672274) · [LMCache × CacheGen blog](https://blog.lmcache.ai/en/2025/07/31/cachegen-store-your-kv-cache-on-disk-or-s3-load-blazingly-fast/)
- [arXiv:2502.15734 — Cache-Craft](https://arxiv.org/abs/2502.15734)
- [arXiv:2511.01815 — KVTC](https://arxiv.org/abs/2511.01815)
- [SIGCOMM 2025 — HACK](https://dl.acm.org/doi/10.1145/3718958.3750481)
- [arXiv:2501.14312 — DLPM / D²LPM](https://arxiv.org/abs/2501.14312)
- [arXiv:2407.11550 — Ada-KV](https://arxiv.org/abs/2407.11550)
- [arXiv:2412.10319 — SCBench](https://arxiv.org/abs/2412.10319)
- [arXiv:2409.04992 — InstInfer](https://arxiv.org/abs/2409.04992)
- [arXiv:2401.02669 — Infinite-LLM / DistAttention](https://arxiv.org/abs/2401.02669)

### Additional context

- [Awesome-KV-Cache-Management list](https://github.com/TreeAI-Lab/Awesome-KV-Cache-Management) — running bibliography of KV cache work.
- [tensormesh.ai — 85% cache hit rate for agents](https://www.tensormesh.ai/blog-posts/agent-skills-caching-cacheblend-llm-cache-hit-rates) — industry writeup, useful numbers but vendor-published.
- [introl.com — prompt caching infra guide](https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025) — aggregated industry numbers for 2025.
