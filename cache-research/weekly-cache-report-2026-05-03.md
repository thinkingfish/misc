# Weekly Cache Research Report — 2026-05-03

**Run type:** Normal weekly run. Prior report: [`weekly-cache-report-2026-04-26.md`](./weekly-cache-report-2026-04-26.md). Search horizon: 2026-04-27 → 2026-05-03 (the seven days following the prior report). Target was ~5 entries; expanded to 8 plus an "outside the window" subsection because the post-ICLR / post-DeepSeek-V4 wave kept landing.

**Scope:** distributed caching, KV cache, caching for inference, storage-system caching.

**Selection criteria:** novelty of mechanism, potential systems / production impact, and whether the work measured runtime efficiency on a realistic workload — flagged explicitly even when the answer is "no" or "partial."

**Organization:** (A) production measurements / empirical studies — real deployments, real traces, real hardware — and (B) academic / idea-forward work — novel mechanisms typically evaluated on benchmarks or simulated traces.

---

## A. Production measurements and empirical studies

### 1. vLLM v0.20.0: DeepSeek-V4 day-0, FA4-default MLA prefill, and TurboQuant 2-bit KV in production (late Apr 2026)

- Reference: [vLLM v0.20.0 release notes](https://github.com/vllm-project/vllm/releases/tag/v0.20.0). Companion: [vLLM blog — DeepSeek V4 in vLLM: Efficient Long-context Attention](https://vllm.ai/blog/deepseek-v4); recipes for [DeepSeek-V4-Pro](https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Pro) and [DeepSeek-V4-Flash](https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash); [release announcement on X](https://x.com/vllm_project/status/2048918629144805619).
- Summary: 752 commits / 320 contributors. Three KV-cache items dominate: (a) **day-0 support for DeepSeek-V4-Pro and -Flash** (1.6T and 285B MoE, 1M-context CSA+HCA hybrid attention from last week's tech report) with first-principles writeups of the attention kernels; (b) **FlashAttention-4 (FA4) re-enabled as the default MLA prefill backend on SM90+ GPUs**; and (c) **TurboQuant 2-bit KV quantization** turned into a real attention backend, advertised as 4× KV capacity at 2 bits with FA3/FA4 prefill, plus an early "vLLM IR" foundation that future kernels (rms_norm and friends) compile against.
- Novelty: Low–medium for the individual mechanisms — FA4, MLA, TurboQuant, and DSv4 hybrid attention each shipped as research artifacts in the previous weeks. High as a packaging event: this is the moment a research stack (TurboQuant from ICLR 2026, CSA/HCA from DeepSeek-V4) becomes the default in the most-deployed open inference engine.
- Impact: Very high. Once 2-bit KV and FA4 are the *default* for new MLA models, the per-GPU economics of long-context serving change for everyone who uses vLLM downstream — Red Hat / llm-d, Together, Fireworks, RunPod, Modal, Anyscale, SageMaker HyperPod. Worth tracking how the "TurboQuant 2-bit, 4× capacity" claim survives continuous-batching tail latency.
- Runtime evaluation: Partial. Headline numbers (4× KV capacity, FA4 prefill speedups) come from the kernel team rather than full SLO traces; the [DeepSeek V4 in vLLM](https://vllm.ai/blog/deepseek-v4) writeup gives an attention-walkthrough but defers production TTFT/TBT distributions to follow-up posts. Independent reproductions are now table-stakes for the next report.

### 2. LMCache — "Stop Calling It KV Cache: It's Something Much Bigger" (Apr 28, 2026)

- Reference: [LMCache Blog — Stop Calling It KV Cache: It's Something Much Bigger](https://blog.lmcache.ai/en/2026/04/28/stop-calling-it-kv-cache-its-something-much-bigger/). Companion: [LMCache on Amazon SageMaker HyperPod (Apr 22, 2026)](https://blog.lmcache.ai/en/2026/04/22/lmcache-on-amazon-sagemaker-hyperpod-accelerating-llm-inference-with-managed-tiered-kv-cache/); [LMCache MoE 10× post (Apr 3, 2026)](https://blog.lmcache.ai/en/2026/04/03/lmcaches-new-architecture-boosts-moe-inference-performance-by-10x/); [LMCache tech report (arXiv:2510.09665)](https://arxiv.org/abs/2510.09665).
- Summary: A position post arguing that "KV cache" — originally a single-request, GPU-resident, ephemeral activation buffer — is no longer an accurate name. With multi-turn chat, agent loops, and long-document RAG forcing prefix reuse across sessions, KV state has become a *first-class data object* with its own lifecycle, multi-tier storage stack (HBM → DRAM → NVMe → networked store), eviction economics, and even cross-tenant sharing semantics. The post is the conceptual companion to LMCache's recent SageMaker HyperPod integration and MoE rearchitecture work.
- Novelty: Low as a technical contribution (no new mechanism), but high as terminology realignment from the team that ships the most-deployed open KV layer. Picks up where the [Modular "Five Eras of KVCache"](https://www.modular.com/blog/the-five-eras-of-kvcache) framing left off and pairs naturally with NVIDIA's CMX / BlueField-4 storage push.
- Impact: Medium-high. Vocabulary matters for buyers — "context memory," "token warehouse," and "KV storage" framing is what justifies VAST / Pure / WEKA selling KV-cache appliances rather than just SSDs. Watch for this language to spread into PR copy across NVIDIA, VAST, Pure, and WEKA in Q2 2026.
- Runtime evaluation: None directly in this post. The companion Apr 22 post on SageMaker HyperPod reports **1.67× ITL improvement and 1.27× higher throughput** for multi-round chat workloads under high concurrency vs. baseline vLLM, on managed tiered storage.

### 3. DeepSeek-V4 technical documentation (released Apr 27, 2026)

- Reference: [DeepSeek V4 model card / technical documentation (PDF)](https://fe-static.deepseek.com/chat/transparency/deepseek-V4-model-card-EN.pdf). Companions: [DeepSeek V4PLUS variant tracking on dev.to (late-April Chinese LLM stack)](https://dev.to/bean_bean/the-late-april-2026-chinese-llm-stack-qwen-36-deepseek-v4plus-kimi-k26-minimax-m27-glm-51-2bif); [NIST CAISI evaluation (May 2026)](https://www.nist.gov/news-events/news/2026/05/caisi-evaluation-deepseek-v4-pro).
- Summary: Followup to the Apr 24 V4-Pro preview (last week's headline entry): DeepSeek published a longer-form **technical documentation** PDF on Apr 27 detailing CSA + HCA hybrid attention, a new **mHC (Manifold-Constrained Hyper-Connections)** residual structure, and a three-mode reasoning framework (Non-think / Think High / Think Max) framed as a first-class architectural feature. A V4PLUS variant ("closed the frontier gap on cost", suitable for RAG over giant documents at 1M context) shipped the same day. NIST CAISI added an external evaluation in May.
- Novelty: Medium-high incremental over last week's report — CSA and HCA were the headline; mHC and the explicit reasoning-mode framework are new in this artifact.
- Impact: High. Reinforces the DeepSeek-V4 KV-cache numbers (~10% of V3.2 KV at 1M tokens, ~27% of V3.2 single-token FLOPs) with primary documentation that downstream stacks (vLLM, SGLang, llm-d) can reference rather than reverse-engineer.
- Runtime evaluation: Inherits from last week's tech report. The new documentation adds NIST-style external evaluation but does not yet contain independent fleet-level latency / TTFT distributions; SGLang's [Day-0 V4 post (Apr 25)](https://www.lmsys.org/blog/2026-04-25-deepseek-v4/) and vLLM v0.20.0 (entry #1) are now the public-evaluation surface.

---

## B. Academic / idea-forward work

### 4. CacheFlow: 3D-Parallel KV Cache Restoration for Long-Context LLM Serving (Apr 28, 2026)

- Reference: Nian, Fang, Feng, Wu, Lai (UIUC / NUS), [arXiv:2604.25080](https://arxiv.org/abs/2604.25080).
- Summary: Reframes KV-cache restoration in distributed serving as a **3D parallelism problem** across tokens, layers, and GPUs — rather than the per-request "recompute vs. fetch" tradeoff that LMCache / CacheGen / CacheBlend implicitly take. A batch-aware two-pointer scheduler picks the operation with the highest marginal reduction in recomputation cost at each step, fine-grained-overlapping I/O and recompute along transformer dependency chains. Implemented atop vLLM + LMCache.
- Novelty: High. Most prior work treats restoration as a *local* decision per request; CacheFlow makes it a global scheduling problem and exploits the structure of transformer dependencies to overlap I/O with compute across requests in a batch.
- Impact: High for distributed serving stacks already running LMCache or analogous KV transport (llm-d, Dynamo). Slots in below the cache layer rather than replacing it.
- Runtime evaluation: Yes — **10–62% TTFT reduction** over existing baselines across multiple models, workloads, and hardware, on top of vLLM + LMCache. The paper does not give detailed continuous-batching tail-latency curves; that's the obvious next benchmark.

### 5. PolyKV: A Shared Asymmetrically-Compressed KV Cache Pool for Multi-Agent LLM Inference (Apr 27, 2026)

- Reference: Patel, Joshi, [arXiv:2604.24971](https://arxiv.org/abs/2604.24971).
- Summary: For multi-agent inference (N agents over a shared context — debate, voting, tool-rollout systems), PolyKV writes the shared context's KV cache **once** in compressed form and injects it into N independent agent contexts via HuggingFace `DynamicCache`. Compression is **asymmetric**: keys at int8 to keep softmax stable, values via TurboQuant-MSE (Walsh-Hadamard rotation + 3-bit Lloyd-Max quantization tuned for N(0,1)).
- Novelty: Medium-high. The multi-agent shared-prefix scenario is well known, but treating the *compressed pool* — not just the prefix — as the deduplication unit, and using asymmetric K/V quantization aligned to softmax sensitivity, is a clean recombination of TurboQuant (entry #4 in the prior report) with prefix sharing.
- Impact: High for agentic / debate / self-consistency / tree-search workloads where N can be 8–32. Less impact when N=1.
- Runtime evaluation: Yes on memory; partial on latency. Llama-3-8B with **15 agents on a 4K-token shared context: 19.8 GB → 0.45 GB KV (97.7% reduction)**, +0.57% perplexity, BERTScore F1 0.928. 2.91× compression ratio holds across SmolLM2-1.7B and Llama-3-8B at 600–7,194 tokens. End-to-end TTFT / TBT under continuous batching not reported.

### 6. CapKV / IB-eviction: Rethinking KV Cache Eviction via a Unified Information-Theoretic Objective (Apr 28, 2026)

- Reference: [arXiv:2604.25975](https://arxiv.org/abs/2604.25975).
- Summary: Casts KV-cache eviction as the **Information Bottleneck** problem: under a linear–Gaussian surrogate of attention, the authors derive a closed-form mutual-information objective for the retained subset. Many existing eviction heuristics (H2O, SnapKV, attention-score eviction) fall out as approximations. **CapKV** is the resulting principled method: log-determinant approximation via statistical leverage scores, no hand-tuned scores or training.
- Novelty: High. The first reasonably general theoretical objective that *unifies* the post-2023 eviction literature instead of adding yet another heuristic. Sits naturally alongside last week's depth-cache lower bound ([arXiv:2604.17935](https://arxiv.org/abs/2604.17935)) — that paper bounds what eviction *cannot* do; this one prescribes what it *should* do.
- Impact: Medium-high near term. Eviction methods will be re-derived against CapKV's objective; expect several follow-up "cheaper approximation of the IB bound" papers within a quarter.
- Runtime evaluation: Partial. Quality benchmarks (multiple models × multiple long-context tasks) showing CapKV outperforms prior eviction at the same memory budget. Kernel-level / TTFT measurements not reported in the abstract — leverage-score computation must be cheap to be paged-attention-friendly, and that integration is not yet shown.

### 7. Predictive Multi-Tier Memory Management for KV Cache in Large-Scale GPU Inference (late Apr 2026)

- Reference: Ganjihal, [arXiv:2604.26968](https://arxiv.org/abs/2604.26968).
- Summary: Proposes a unified KV-cache subsystem that combines (a) **architecture-aware sizing** for MHA / GQA / MQA / MLA (claims up to 57× memory over-provisioning in current general-purpose frameworks for MLA), (b) a **six-tier memory hierarchy** spanning HBM → DRAM → CXL-attached memory → GPUDirect Storage NVMe → RDMA fabric → parallel filesystem, and (c) **predictive eviction** that models reuse rather than reacting to it. Combines the three ideas that prior systems address one at a time.
- Novelty: Medium-high. None of the three pieces is individually new — Mooncake / LMCache / Dynamo each cover one tier-class — but the *unified* architecture-variant sizing (especially explicit MLA support) and the predictive eviction layer are the new content.
- Impact: Potentially high if validated. This is the "one-stop" framing that NVIDIA's CMX / Inference Context Memory Storage Platform (ICMSP) is building toward; if reproduced, it gives a clean reference design for partner storage stacks (VAST, Pure, WEKA).
- Runtime evaluation: **Analytical projections, not measurements.** On a 64-GPU H100 cluster, projects 4,150 tokens/s/GPU at $0.43/M tokens, 2.0× TensorRT-LLM throughput at 30% lower cost, 6.4× FlexGen throughput, 11× lower TTFT P99. The "projection vs. measurement" gap is the paper's main weakness — needs an empirical follow-up before any production claim is real.

### 8. Dual-Blade: Dual-Path NVMe-Direct KV-Cache Offloading for Edge LLM Inference (Apr 29, 2026)

- Reference: Jeong, Byun, Y. Kim, Yu, K. Lee, Yang, Park, [arXiv:2604.26557](https://arxiv.org/abs/2604.26557).
- Summary: Edge / on-device inference where KV exceeds available memory. Most NVMe offloading goes through the kernel page cache, which thrashes under memory pressure. Dual-Blade keeps the page-cache path *and* a **kernel-bypass NVMe-direct path** that maps KV tensors to contiguous LBA regions, then dynamically routes per-tensor based on runtime memory headroom — the storage analogue of SparKV's wireless-aware compute-or-load (last week's edge entry).
- Novelty: Medium. NVMe-direct / GPUDirect Storage is well known on the datacenter side; the mechanism here is the *dynamic dual-path arbitration*, taking memory-pressure as the routing signal at edge scale.
- Impact: Medium-high for the edge-inference subfield (joining KVSwap, SparKV, "Don't Waste Bits!" from prior reports). Less directly relevant to datacenter serving, where DRAM and CXL are usually plentiful and the page cache is rarely the bottleneck.
- Runtime evaluation: Yes — head-to-head against page-cache offloading on edge hardware under memory pressure, claiming reduced thrashing, more predictable latency, and lower software overhead. Absolute TTFT / energy numbers not summarised in the abstract but the methodology is the right one for this niche.

---

## A′. Closely related work just outside the 7-day window

### 9. Cloudflare — "Building the foundation for running extra-large language models" (mid-late Apr 2026)

- Reference: Cloudflare blog, [Building the foundation for running extra-large language models](https://blog.cloudflare.com/high-performance-llms/). Coverage: [InfoQ — Cloudflare Builds High-Performance Infrastructure for Running LLMs (May 2026)](https://www.infoq.com/news/2026/05/cloudflare-llm-infrastructure/).
- Summary: Production engineering post on running ≥1T-parameter open-weight models (e.g. Kimi K2.5 at ~560 GB) on Cloudflare's edge. Uses **prefill–decode disaggregation** (compute-bound prefill server populates KV, then transfers to memory-bound decode server), **NVLink + NVMe-over-Fabric RDMA** for KV transfer, **session-affinity headers** for prompt caching with cached-token discounts, **Mooncake's Transfer Engine** for the KV-transport substrate, and **R2 object storage** for cached model weights. Speculative decoding overlays the whole thing.
- Novelty: Low as mechanism — none of the building blocks is new — but high as the most explicit edge-CDN production reference architecture for cross-machine PD disaggregation. The choice of Mooncake's Transfer Engine in production is interesting (and aligns with last week's PrfaaS direction).
- Impact: High signal value. When Cloudflare publicly endorses Mooncake's transport, NVMe-over-Fabric KV transfer, and session-affinity prefix caching, those are no longer research directions; they are a CDN-tier industry default.
- Runtime evaluation: Architectural rather than measurement-led — describes the stack but does not publish TTFT / tok/s curves under load. (The companion [Unweight tensor-compression post (Apr 20)](https://blog.cloudflare.com/unweight-tensor-compression/) does have a 15–22% lossless weight-compression number, which is adjacent but orthogonal.)

### 10. DepthKV: Layer-Dependent KV Cache Pruning (Apr 27, 2026)

- Reference: Dehghanighobadi, Fischer, [arXiv:2604.24647](https://arxiv.org/abs/2604.24647).
- Summary: Allocates a fixed *global* KV pruning budget non-uniformly across layers based on per-layer sensitivity — supporting position-dependent protection (e.g. preserving middle layers), metric-guided allocation, and hybrid strategies, all under one budget constraint.
- Novelty: Low–medium. Layer-wise KV-budget allocation has been explored (PyramidKV, Adaptive-KV); the contribution is the budget-respecting unified framework and a clean comparison harness.
- Impact: Medium. A useful drop-in for any eviction backbone; complementary to CapKV (entry #6) — CapKV decides *which* tokens, DepthKV decides *how many per layer*.
- Runtime evaluation: Yes — quality benchmarks across multiple models / tasks at matched global pruning ratio, consistently above uniform allocation. Throughput / TTFT numbers not the focus.

---

## Cross-cutting observations (this week)

- **The TurboQuant story closes the loop in seven days.** ICLR 2026 → Triton/vLLM patches → vLLM v0.20.0 default 2-bit KV with 4× capacity (entry #1) is one of the fastest research-to-production paths the KV-cache subfield has seen. The companion data point — vLLM also re-enabled FA4 as the default MLA prefill backend on SM90+ — means that the *defaults* of the most-deployed open inference engine moved this week, not just an optional flag.
- **"KV cache" as a name is officially under pressure.** The LMCache blog (entry #2) makes the rename argument explicit, and DeepSeek-V4's technical documentation (entry #3), VAST / Pure CMX integration, and Cloudflare's Mooncake-Transfer-Engine architecture (entry #9) all treat KV state as a tiered, persistent, shared data object rather than a per-request scratch buffer. Expect vendor pitches in Q2 2026 to converge on "context memory" / "token warehouse" framing.
- **Theory and unification are catching up to mechanism.** CapKV (entry #6) gives the post-2023 eviction literature a unified IB objective; it pairs with last week's [depth–cache lower bound](https://arxiv.org/abs/2604.17935) to form a "what should we keep, and how compressed can we get" theoretical pincer. That conversation now has a real prescriptive theory — not just heuristics.
- **Multi-agent shared-pool KV is its own emerging subgenre.** PolyKV (entry #5) joins last quarter's [KVComm (arXiv:2510.12872)](https://arxiv.org/abs/2510.12872) and the [Persistent Q4 KV (arXiv:2603.04428)](https://arxiv.org/abs/2603.04428) edge-multi-agent line. As agents-as-default takes hold, the "N agents over one context" workload is becoming a distinct optimization target separate from chat / RAG / single-prefix.
- **Storage-as-KV is the largest unresolved measurement gap.** Predictive Multi-Tier (entry #7) is *projection*-only on a 64-H100 cluster; CMX / BlueField-4 / VAST / Pure are vendor-architectural; only LMCache (HyperPod, MoE 10×) and llm-d post real fleet numbers. This week extends the gap rather than closing it; whoever publishes the first independent end-to-end benchmark of a six-tier KV stack against vLLM v0.20.0 + LMCache will set the reference.
- **Edge inference now has its own page-cache story.** Dual-Blade (entry #8) is the second consecutive week with an edge-KV paper, and the second one to treat *kernel storage paths* — not just compression — as the bottleneck. Edge KV is no longer just "compress harder"; it is becoming a small storage-systems problem in its own right.

## References

- [vLLM v0.20.0 release notes](https://github.com/vllm-project/vllm/releases/tag/v0.20.0) · [vLLM blog — DeepSeek V4 in vLLM](https://vllm.ai/blog/deepseek-v4) · [vLLM Recipes: DeepSeek-V4-Pro](https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Pro) · [vLLM Recipes: DeepSeek-V4-Flash](https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash) · [vLLM v0.20.0 announcement (X/Twitter)](https://x.com/vllm_project/status/2048918629144805619) · [SGLang Day-0 DeepSeek-V4 (LMSYS, Apr 25, 2026)](https://www.lmsys.org/blog/2026-04-25-deepseek-v4/)
- [LMCache — Stop Calling It KV Cache (Apr 28, 2026)](https://blog.lmcache.ai/en/2026/04/28/stop-calling-it-kv-cache-its-something-much-bigger/) · [LMCache on Amazon SageMaker HyperPod (Apr 22, 2026)](https://blog.lmcache.ai/en/2026/04/22/lmcache-on-amazon-sagemaker-hyperpod-accelerating-llm-inference-with-managed-tiered-kv-cache/) · [LMCache MoE 10× (Apr 3, 2026)](https://blog.lmcache.ai/en/2026/04/03/lmcaches-new-architecture-boosts-moe-inference-performance-by-10x/) · [LMCache tech report (arXiv:2510.09665)](https://arxiv.org/abs/2510.09665)
- [DeepSeek V4 Technical Documentation (Apr 27, 2026, PDF)](https://fe-static.deepseek.com/chat/transparency/deepseek-V4-model-card-EN.pdf) · [DeepSeek V4 Preview API release note (Apr 24, 2026)](https://api-docs.deepseek.com/news/news260424) · [DeepSeek-V4-Pro (Hugging Face)](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro) · [Late-April Chinese LLM stack overview (DEV)](https://dev.to/bean_bean/the-late-april-2026-chinese-llm-stack-qwen-36-deepseek-v4plus-kimi-k26-minimax-m27-glm-51-2bif) · [NIST CAISI Evaluation of DeepSeek V4 Pro (May 2026)](https://www.nist.gov/news-events/news/2026/05/caisi-evaluation-deepseek-v4-pro)
- [arXiv:2604.25080 — CacheFlow (Apr 28, 2026)](https://arxiv.org/abs/2604.25080)
- [arXiv:2604.24971 — PolyKV (Apr 27, 2026)](https://arxiv.org/abs/2604.24971)
- [arXiv:2604.25975 — CapKV / Information-Theoretic KV Eviction (Apr 28, 2026)](https://arxiv.org/abs/2604.25975)
- [arXiv:2604.26968 — Predictive Multi-Tier Memory Management (late Apr 2026)](https://arxiv.org/abs/2604.26968)
- [arXiv:2604.26557 — Dual-Blade NVMe-Direct KV-Cache Offloading (Apr 29, 2026)](https://arxiv.org/abs/2604.26557)
- [Cloudflare — Building the foundation for running extra-large language models](https://blog.cloudflare.com/high-performance-llms/) · [InfoQ coverage (May 2026)](https://www.infoq.com/news/2026/05/cloudflare-llm-infrastructure/) · [Cloudflare — Unweight tensor compression (Apr 20, 2026)](https://blog.cloudflare.com/unweight-tensor-compression/)
- [arXiv:2604.24647 — DepthKV (Apr 27, 2026)](https://arxiv.org/abs/2604.24647)

### Additional context (noted this week, not selected as top entries)

- [arXiv:2604.20289 — X-Cache (cross-chunk block caching for autoregressive driving world models, Apr 22, 2026)](https://arxiv.org/abs/2604.20289) — caches across generation chunks rather than denoising steps; KV-cache-shaped but in the world-model / video-DiT regime. Just outside the LLM-inference scope.
- [arXiv:2604.22782 — Stochastic KV Routing (depth-wise cache sharing, Apr 3, 2026)](https://arxiv.org/abs/2604.22782) — random cross-layer attention during training enables runtime depth-wise cache sharing; an orthogonal axis to temporal compression.
- [arXiv:2604.19351 — DASH-KV (asymmetric KV-cache hashing, ACL 2026 Findings)](https://arxiv.org/abs/2604.19351) — recasts attention as approximate-NN search over an asymmetric hash; complements rather than replaces eviction/quantization.
- [arXiv:2604.20021 — Continuous Semantic Caching for low-cost LLM serving](https://arxiv.org/abs/2604.20021) — first rigorous regret-bounded framework for *response*-level semantic caching in continuous embedding space; outside the KV-cache subfield but shares the prefix-reuse economics.
- [Modular — The Five Eras of KVCache](https://www.modular.com/blog/the-five-eras-of-kvcache) — historical / framing companion to LMCache's "Stop calling it KV cache" post.
- [Awesome-KV-Cache-Management (TreeAI-Lab)](https://github.com/TreeAI-Lab/Awesome-KV-Cache-Management) and [Awesome-KV-Cache-Optimization (jjiantong, ACL 2026 survey)](https://github.com/jjiantong/Awesome-KV-Cache-Optimization) — useful trackers updated through this window.
