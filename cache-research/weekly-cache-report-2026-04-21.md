# Weekly Cache Research Report — 2026-04-21

**Run type:** Catch-up (first run — no prior reports found). Search horizon expanded to the last ~12 months; top 20 entries selected.

**Scope:** distributed caching, KV cache, caching for inference, storage-system caching.

**Selection criteria:** novelty of mechanism, potential systems/production impact, and whether the work actually measured runtime efficiency (latency, throughput, memory, I/O, SLO) on a realistic workload rather than only model-quality metrics.

---

## 1. Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving (FAST '25 Best Paper)

- Reference: Qin et al., FAST '25 / arXiv:2407.00079. https://arxiv.org/abs/2407.00079 — https://www.usenix.org/conference/fast25/presentation/qin
- Summary: Disaggregates prefill and decode clusters and pools CPU DRAM, SSD, and RDMA fabric across the GPU cluster into a shared "KVCache pool." An early-rejection scheduler protects SLOs under overload.
- Novelty: High. Reframes LLM serving as a storage system problem ("trade storage for compute") and publishes a production trace from Kimi. Open-sourced Transfer Engine (Nov 2024) and Mooncake Store (Mar 2025).
- Impact: Very high. Adopted/influenced vLLM, SGLang, llm-d; cited as the reference disaggregated design. Reported 525% throughput improvement on long-context workloads under simulation of real traces.
- Runtime evaluation: Yes — TTFT, TBT, end-to-end throughput, SLO attainment on Kimi production traces with H800-class fleets; uses real RDMA interconnect and staged DRAM/SSD.

## 2. LMCache: An Efficient KV Cache Layer for Enterprise-Scale LLM Inference

- Reference: Liu et al., arXiv:2510.09665, Oct 2025. https://arxiv.org/abs/2510.09665
- Summary: Cross-engine KV cache layer that sits outside GPU memory, supporting prefix and non-prefix reuse across vLLM/SGLang instances, with batched data movement and compute-I/O pipelining behind a modular connector.
- Novelty: Medium-high. Generalizes "prefix caching" to engine-agnostic, tiered (GPU→CPU→SSD→remote) cache with a production API; less about a single algorithm and more about a reusable substrate.
- Impact: High. Becoming the de-facto open-source KV cache middleware for enterprise inference stacks; referenced by llm-d as its cache layer.
- Runtime evaluation: Yes — measured TTFT, throughput, and prefix hit ratio under enterprise trace workloads; explicitly reports I/O pipelining wins.

## 3. DualPath: Breaking the Storage Bandwidth Bottleneck in Agentic LLM Inference

- Reference: Wu et al., arXiv:2602.21548 (Feb 2026). https://hf.co/papers/2602.21548
- Summary: For multi-turn/agentic serving, splits KV-cache loads into two paths (direct RDMA fetch vs. local recompute) and does dynamic load balancing across prefill/decode engines using a global scheduler.
- Novelty: High. Recognizes that in agentic workloads the bottleneck shifts from compute to KV storage bandwidth and schedules against that dimension.
- Impact: Potentially high for long-running agents and tool-using LLM services; 50 upvotes and active discussion.
- Runtime evaluation: Yes — offline and online serving benchmarks; RDMA bandwidth utilization and per-turn latency.

## 4. DualMap: Enabling Both Cache Affinity and Load Balancing for Distributed LLM Serving

- Reference: Yuan et al., arXiv:2602.06502 (Feb 2026). https://hf.co/papers/2602.06502
- Summary: Two independent hash rings route requests by cache affinity and by load simultaneously, with SLO-aware hotspot rebalancing and a dual-hash-ring scaling scheme.
- Novelty: Medium-high. Attacks the classic tension between locality (prefix-cache hit) and balancing — usually treated as a single scheduling knob — with two decoupled rings.
- Impact: Medium-high for multi-tenant, multi-replica deployments where a few hot prefixes skew utilization.
- Runtime evaluation: Yes — SLO attainment and goodput under skewed synthetic and real traces.

## 5. CONCUR: High-Throughput Agentic Batch Inference via Congestion-Based Concurrency Control

- Reference: Chen et al., arXiv:2601.22705 (Jan 2026). https://hf.co/papers/2601.22705
- Summary: Treats middle-phase KV-cache thrashing in agentic batch inference like TCP congestion — an adaptive agent-admission control layer using cache-occupancy signals.
- Novelty: High. Transfers a classic networking idea (congestion control) to LLM cache scheduling, framing thrashing as a solvable control problem rather than a resource-sizing one.
- Impact: Medium-high for batch/agent offline workloads where thrashing collapses goodput.
- Runtime evaluation: Yes — goodput and tail latency under admission control vs. baselines on mixed agentic traces.

## 6. Locality-aware Fair Scheduling in LLM Serving (DLPM/D²LPM)

- Reference: Cao et al., arXiv:2501.14312, Jan 2025. https://arxiv.org/abs/2501.14312
- Summary: Introduces Deficit Longest Prefix Match (and Double Deficit LPM) to trade prefix-cache locality for per-tenant fairness without sacrificing throughput.
- Novelty: Medium-high. The Virtual Token Counter + LPM combination cleanly models a three-way objective (fairness, locality, balance).
- Impact: High for multi-tenant serving where one tenant's hot prefixes can starve others.
- Runtime evaluation: Yes — throughput, p99 latency, fairness index across tenants on synthetic and real traces.

## 7. KV Cache Transform Coding for Compact Storage (KVTC)

- Reference: Staniszewski & Łańcucki, arXiv:2511.01815, Nov 2025. https://arxiv.org/abs/2511.01815
- Summary: PCA-based decorrelation + adaptive quantization + entropy coding applied to K/V, giving up to 20× compression of reusable KV caches while preserving accuracy.
- Novelty: Medium-high. Revives classical transform-coding tools (image-codec style) for KV, and targets *stored* caches rather than online attention.
- Impact: High for any tiered KV cache (LMCache / Mooncake Store) where cache size and bandwidth dominate cost.
- Runtime evaluation: Partial — accuracy vs. compression ratio is thorough; end-to-end serving latency is less emphasized than storage reduction.

## 8. IceCache: Memory-efficient KV-cache Management for Long-Sequence LLMs

- Reference: Mao et al., arXiv:2604.10539, Apr 2026. https://hf.co/papers/2604.10539
- Summary: Combines semantic token clustering with a PagedAttention-compatible hierarchical structure to cut CPU↔GPU KV transfers for long sequences.
- Novelty: Medium. Clustering + paging is increasingly common; the contribution is the integration and selection policy.
- Impact: Medium for long-context, memory-constrained inference.
- Runtime evaluation: Yes — LongBench accuracy alongside memory footprint and transfer volume.

## 9. EvolKV: Evolutionary KV Cache Compression

- Reference: Yu & Chai, arXiv:2509.08315, Sep 2025. https://arxiv.org/abs/2509.08315
- Summary: Evolutionary search optimizes per-layer KV budget allocation against downstream task loss, producing non-uniform compression profiles.
- Novelty: Medium. The per-layer budget-search framing is clean and task-driven; evolutionary search is pragmatic.
- Impact: Medium. Useful as a tuning layer over Ada-KV / SnapKV-style eviction.
- Runtime evaluation: Partial — GSM8K and code completion quality vs. memory; limited deployment-latency data.

## 10. Ada-KV: Adaptive Budget Allocation for KV Cache Eviction

- Reference: Feng et al., arXiv:2407.11550 (updated Oct 2025). https://arxiv.org/abs/2407.11550
- Summary: Allocates different per-head eviction budgets using attention-pattern statistics rather than a uniform budget.
- Novelty: Medium. Important empirical observation (heads differ) turned into a clean allocation rule; now a widely cited baseline.
- Impact: High — adopted as a baseline and integrated into several KV-compression pipelines.
- Runtime evaluation: Yes — memory vs. quality across LongBench; limited throughput measurements.

## 11. KV Admission: Learning What to Write for Efficient Long-Context Inference

- Reference: Huang et al., arXiv:2512.17452, Dec 2025. https://hf.co/papers/2512.17452
- Summary: Adds a "write gate" before cache entry, learned from token utility — a causal complement to the more common post-hoc eviction and selection.
- Novelty: Medium-high. Moves the compression decision *earlier* in the pipeline; integrates with FlashAttention / Paged-KV.
- Impact: Medium. Could stack with eviction policies; useful when write bandwidth (HBM or DRAM) is the bottleneck.
- Runtime evaluation: Yes — prefill/decode latency, memory savings on long-context benchmarks.

## 12. TailorKV: Hybrid Quantization + Offloading for Long Context

- Reference: Yao et al., arXiv:2505.19586, May 2025. https://arxiv.org/abs/2505.19586
- Summary: Splits tokens into "global info" (offload) and "dominant activations" (keep on GPU, quantized) to balance PCIe bandwidth against GPU memory.
- Novelty: Medium. The partitioning rule is the main contribution; quantization and offloading are established.
- Impact: Medium — targets single-GPU long-context (RTX 3090 class).
- Runtime evaluation: Yes — end-to-end latency on commodity GPUs with PCIe as the bottleneck.

## 13. Cache-Craft: Managing Chunk-Caches for Efficient RAG

- Reference: Agarwal et al., arXiv:2502.15734, Feb 2025. https://arxiv.org/abs/2502.15734
- Summary: For RAG, reuses precomputed per-chunk KV across different queries/orders, handling positional and cross-chunk attention carefully.
- Novelty: High. Cross-query reuse of chunk KVs (not just prefix) is tricky; the paper makes it safe enough to ship.
- Impact: High for RAG-heavy production workloads where retrieved chunk set is often overlapping.
- Runtime evaluation: Yes — TTFT, throughput, quality parity vs. vLLM on LLaMA-3-8B/70B.

## 14. RAGBoost: Accuracy-Preserving Context Reuse for RAG

- Reference: Jiang et al., arXiv:2511.03475, Nov 2025. https://arxiv.org/abs/2511.03475
- Summary: Improves RAG prefill caching via context indexing, ordering, de-duplication, and "contextual hints" that preserve reasoning fidelity.
- Novelty: Medium-high. Complements Cache-Craft by focusing on what re-ordering/de-dup does to reasoning quality.
- Impact: Medium-high for agentic RAG pipelines.
- Runtime evaluation: Yes — prefill latency and accuracy on RAG benchmarks with multiple inference engines.

## 15. RAGPulse: An Open-Source RAG Workload Trace

- Reference: Wang et al., arXiv:2511.12979, Nov 2025. https://arxiv.org/abs/2511.12979
- Summary: Real-world trace of a Q&A RAG system revealing temporal locality and hot-document patterns; intended as a benchmark for cache / batching research.
- Novelty: Medium — the trace itself is the contribution, but it fills a real gap.
- Impact: High enabler for future work — comparable to the role the Mooncake trace plays for serving.
- Runtime evaluation: N/A (trace/dataset paper); provides the substrate to evaluate other systems.

## 16. Category-Aware Semantic Caching for Heterogeneous LLM Workloads

- Reference: Wang et al., arXiv:2510.26835, Oct 2025. https://arxiv.org/abs/2510.26835
- Summary: Adapts similarity threshold, TTL, and quota per *query category* over HNSW, rather than a global threshold that is either too loose or too strict.
- Novelty: Medium. Addresses a well-known failure mode of semantic caches (one-size threshold) with a clean workload-aware fix.
- Impact: Medium-high for public LLM APIs with diverse traffic (chat, code, safety, tools).
- Runtime evaluation: Yes — hit rate, coverage, and false-hit rate on heterogeneous query mixes.

## 17. SCBench: A KV Cache-Centric Analysis of Long-Context Methods

- Reference: Li et al., arXiv:2412.10319, Dec 2024. https://arxiv.org/abs/2412.10319
- Summary: Benchmark that evaluates long-context methods along the four KV lifecycle stages — generation, compression, retrieval, loading — instead of end-task only.
- Novelty: High. Exposes that many long-context methods win on one stage and lose on another; the "layer-level sparsity" finding is cited widely.
- Impact: High as a measurement framework for KV research; already a standard reference.
- Runtime evaluation: Yes — this *is* the runtime evaluation framework; measures attention distribution shift and stage-wise cost.

## 18. Infinite-LLM / DistAttention: Distributed KVCache for Long Context

- Reference: Lin et al., arXiv:2401.02669 (Jan 2024, still referenced through 2025). https://arxiv.org/abs/2401.02669
- Summary: DistAttention partitions attention across nodes and pools KV across CPU/GPU in a cloud fleet, aimed at elastic context growth.
- Novelty: Medium-high at publication; foundational for the later Mooncake / LMCache wave.
- Impact: High as a predecessor architecture for the disaggregated-KV trend.
- Runtime evaluation: Yes — end-to-end throughput, context-length reachability in a data-center setting.

## 19. InstInfer: In-Storage Attention Offloading on Computational Storage Drives

- Reference: Pan et al., arXiv:2409.04992, Sep 2024. https://arxiv.org/abs/2409.04992
- Summary: Offloads attention compute *and* KV to Computational Storage Drives, exploiting the drive's internal bandwidth and P2P transfers to bypass host PCIe.
- Novelty: High. Crosses the hardware boundary — attention runs inside the SSD controller — and is one of the few papers truly on the storage-system-caching frontier.
- Impact: Medium today, potentially high if CSDs become commodity; most relevant for cost-constrained long-context inference.
- Runtime evaluation: Yes — decoding throughput vs. host-offloaded baselines; PCIe bandwidth saturation analysis.

## 20. Optimizing SSD Caches for Cloud Block Storage Using ML

- Reference: Wang et al., arXiv:2501.14770, Jan 2025. https://arxiv.org/abs/2501.14770
- Summary: ML-driven write-policy optimization for SSD caches in cloud block storage: predicts cold writes to skip the flash layer, improving hit rate and endurance. Sits alongside CMU's Baleen (CMU-CS-24-152) line of ML-for-flash-cache work.
- Novelty: Medium. Builds on a rich ML-admission literature; contribution is the production-trace evaluation and write-skip policy.
- Impact: High for cloud storage operators (endurance is a real cost driver); outside the LLM bubble but directly relevant to "caching" as a systems problem.
- Runtime evaluation: Yes — hit rate, write amplification, SSD endurance on block-storage traces.

---

## Cross-cutting observations

- **KV cache is the new storage tier.** Five of the top entries (Mooncake, LMCache, DualPath, DualMap, Infinite-LLM) treat KV cache as a first-class distributed-storage problem — pooled DRAM/SSD over RDMA, hash-ring placement, SLO-aware scheduling. This is the clearest trend of the past year.
- **Control-theoretic framings are appearing.** CONCUR borrows TCP congestion control; Locality-aware Fair Scheduling borrows deficit round-robin. Pure heuristics are giving way to analyzable policies.
- **Compression is shifting from eviction to coding.** KVTC (transform coding), EvolKV (search-optimized budgets), KV Admission (write-gating) move beyond the 2024 wave of pure eviction heuristics.
- **RAG caching is maturing.** Cache-Craft, RAGBoost, and RAGPulse together indicate RAG-specific cache reuse (not just prefix) is becoming a subfield with its own benchmarks.
- **Runtime-efficiency reporting is uneven.** System-venue papers (Mooncake, LMCache, DualPath, Infinite-LLM) report full TTFT/TBT/SLO attainment on realistic traces. Many ML-venue KV-compression papers still report only quality vs. memory and skip serving-latency numbers — a gap the field should close.
- **Storage-system-caching research continues independently** (ML for SSD admission/prefetching, columnar-format random access like Lance) and has so far seen limited cross-pollination with the LLM-KV wave despite the obvious analogies.

## Sources

- https://arxiv.org/abs/2407.00079 — Mooncake (FAST '25)
- https://www.usenix.org/conference/fast25/presentation/qin — Mooncake FAST '25 page
- https://github.com/kvcache-ai/Mooncake — Mooncake open-source
- https://arxiv.org/abs/2510.09665 — LMCache
- https://lmcache.ai/tech_report.pdf — LMCache tech report
- https://hf.co/papers/2602.21548 — DualPath
- https://hf.co/papers/2602.06502 — DualMap
- https://hf.co/papers/2601.22705 — CONCUR
- https://arxiv.org/abs/2501.14312 — Locality-aware Fair Scheduling
- https://arxiv.org/abs/2511.01815 — KVTC
- https://hf.co/papers/2604.10539 — IceCache
- https://arxiv.org/abs/2509.08315 — EvolKV
- https://arxiv.org/abs/2407.11550 — Ada-KV
- https://hf.co/papers/2512.17452 — KV Admission
- https://arxiv.org/abs/2505.19586 — TailorKV
- https://arxiv.org/abs/2502.15734 — Cache-Craft
- https://arxiv.org/abs/2511.03475 — RAGBoost
- https://arxiv.org/abs/2511.12979 — RAGPulse
- https://arxiv.org/abs/2510.26835 — Category-Aware Semantic Caching
- https://arxiv.org/abs/2412.10319 — SCBench
- https://arxiv.org/abs/2401.02669 — Infinite-LLM / DistAttention
- https://arxiv.org/abs/2409.04992 — InstInfer
- https://arxiv.org/abs/2501.14770 — ML for SSD caches in cloud block storage
- https://llm-d.ai/blog/kvcache-wins-you-can-see — llm-d prefix caching blog
- https://docs.vllm.ai/en/stable/design/prefix_caching/ — vLLM prefix caching
- https://redis.io/blog/what-is-semantic-caching/ — Redis semantic caching
- https://www.frontiersin.org/journals/computer-science/articles/10.3389/fcomp.2025.1511161/full — Distributed caching with strong consistency (2025)
- http://reports-archive.adm.cs.cmu.edu/anon/2024/CMU-CS-24-152.pdf — Baleen / ML for flash caching (CMU thesis)
