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
