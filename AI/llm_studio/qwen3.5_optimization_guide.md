# Qwen 3.5 9B (Q8_0) Configuration & Optimization Guide

*Tailored for ASUS ExpertBook P3606CCAP (Intel Core Ultra 7 255H, 24 GB RAM, Intel Arc 140T iGPU)*

---

## 1. Context & Hardware Reality Check

### Model Profile
* **Model Identifier:** `qwen/qwen3.5-9b`
* **File:** `Qwen3.5-9B-Q8_0.gguf`
* **Disk Size:** 10.45 GB
* **Quantization:** Q8_0 (8-bit quantization; maximum precision available for GGUF before FP16, but requires significant RAM/VRAM).

### Hardware Limitations
* **RAM/VRAM Pool:** 24 GB DDR5 total RAM. Your Intel Arc 140T iGPU shares this system memory pool.
* **Available Free RAM:** ~10–18 GB depending on background tasks.
* **Memory Math:** Loading a 10.45 GB weights file leaves roughly 7–10 GB of free RAM. An uncompressed context length of 128,000 tokens requires **over 16–32 GB of KV Cache RAM alone**, which **will crash or heavily swap** your system if left unoptimized.

---

## 2. Parameter-by-Parameter Field Analysis & Recommendations

### A. Load Settings (`Load` Tab)

#### 1. Context Length
* **Current Setting:** `128,000`
* **Suggested Setting:** `16,384` to `32,768` (16k–32k)
* **Explanation & Reason:** 
  * `Context Length` determines how many total tokens (prompt + output) the model can hold in memory simultaneously.
  * *Why change it?* Setting this to 128,000 tokens requires allocating gigabytes of system memory for the Key-Value (KV) cache. At 128k, KV cache memory footprint far exceeds your available 24 GB system RAM when combined with the 10.45 GB model file. Setting it to 16,384 or 32,768 keeps total RAM usage safely under 18 GB while maintaining ample context for long conversations and document analysis.

#### 2. GPU Offload
* **Current Setting:** `32` layers
* **Suggested Setting:** Max slider value (offload all layers possible, usually 32 or 36 depending on architecture)
* **Explanation & Reason:**
  * Controls how many neural network layers are transferred from the CPU to your GPU for execution.
  * *Why keep/increase it?* Your Intel Arc 140T GPU processes matrix multiplications significantly faster than the CPU via Vulkan or OpenVINO backends. Since Arc 140T uses shared system memory, offloading all layers utilizes unified memory bandwidth while offloading execution tasks to the iGPU compute units.

---

### B. Advanced Load Parameters (`Advanced` Section)

#### 3. CPU Thread Pool Size
* **Current Setting:** `6`
* **Suggested Setting:** `6` (Keep as is)
* **Explanation & Reason:**
  * Sets the maximum number of CPU worker threads used during prompt processing and CPU fallback execution.
  * *Why 6?* Your Core Ultra 7 255H CPU has 6 Performance (P) cores. Using 6 threads maps 1:1 with high-performance physical cores, preventing latency spikes caused by sending inference work to slower Efficiency (E) or Low-Power Efficiency (LPE) cores.

#### 4. Evaluation Batch Size & Physical Batch Size
* **Current Setting:** `Evaluation Batch Size: 2048` | `Physical Batch Size: 512`
* **Suggested Setting:** `Evaluation Batch Size: 512` | `Physical Batch Size: 512`
* **Explanation & Reason:**
  * `Evaluation Batch Size` (Prompt processing batch size) dictates how many tokens are ingested at once during initial prompt reading.
  * `Physical Batch Size` sets the chunk size sent to hardware per pass.
  * *Why change?* Reducing Evaluation Batch Size to 512 reduces peak RAM spikes during prompt ingestion without noticeably reducing processing speed on integrated GPUs.

#### 5. Max Concurrent Predictions (Experimental)
* **Current Setting:** `4`
* **Suggested Setting:** `1`
* **Explanation & Reason:**
  * Defines how many parallel requests the local server engine processes simultaneously.
  * *Why 1?* Local consumer devices (especially with shared memory iGPUs) suffer severe performance degradation when processing parallel streams. Set to 1 for dedicated personal assistant use.

#### 6. Unified KV Cache (Experimental)
* **Current Setting:** `Enabled (On)`
* **Suggested Setting:** `Enabled (On)`
* **Explanation & Reason:**
  * Manages key-value memory blocks in a single contiguous pool for better memory efficiency. Keep enabled.

#### 7. Context Checkpoints
* **Current Setting:** `32`
* **Suggested Setting:** `32` (Default)
* **Explanation & Reason:**
  * Used for state caching and context reuse. Leaving at default works well.

#### 8. RoPE Frequency Base & Scale
* **Current Setting:** `Auto` / `Unchecked`
* **Suggested Setting:** `Auto` / `Unchecked`
* **Explanation & Reason:**
  * Controls Rotary Position Embedding scaling for extended context. Leave on `Auto` as GGUF metadata automatically supplies native RoPE parameters for Qwen 3.5.

#### 9. Offload KV Cache to GPU Memory
* **Current Setting:** `Disabled (Off)`
* **Suggested Setting:** `Enabled (On)`
* **Explanation & Reason:**
  * Determines whether the active KV context cache is stored in iGPU memory space or pure host RAM.
  * *Why enable?* Since your Arc 140T handles GPU offloaded layers, storing the KV cache in GPU-accessible memory prevents bandwidth bottlenecking between the CPU and iGPU during token generation.

#### 10. Keep Model in Memory
* **Current Setting:** `Enabled (On)`
* **Suggested Setting:** `Enabled (On)`
* **Explanation & Reason:**
  * Prevents Windows from unloading model weights from RAM when idle. Ensures immediate response times without re-loading 10.45 GB from SSD every prompt.

#### 11. Try mmap()
* **Current Setting:** `Enabled (On)`
* **Suggested Setting:** `Enabled (On)`
* **Explanation & Reason:**
  * Uses Memory-Mapped I/O to load model files directly into RAM virtual address space. Enables fast loading and efficient memory management.

#### 12. Seed
* **Current Setting:** `Unchecked (Random Seed)`
* **Suggested Setting:** `Unchecked` for general use; set a specific number only for reproducible benchmark tests.

#### 13. Speculative Decoding
* **Current Setting:** `Off`
* **Suggested Setting:** `Off`
* **Explanation & Reason:**
  * Requires loading a second, smaller "draft" model alongside Qwen 3.5 9B. On a 24 GB setup with a Q8 model, loading a second draft model risks exceeding memory limits.

#### 14. Flash Attention
* **Current Setting:** `Enabled (On)`
* **Suggested Setting:** `Enabled (On)`
* **Explanation & Reason:**
  * Drastically optimizes memory bandwidth and computation for attention matrices. **Critical for keeping context RAM overhead low.**

#### 15. K Cache / V Cache Quantization Type (Experimental)
* **Current Setting:** `Unchecked` (FP16 full precision)
* **Suggested Setting:** `K Cache: Q8_0` or `Q4_0` | `V Cache: Q8_0` or `Q4_0`
* **Explanation & Reason:**
  * Quantizes key-value context vectors in RAM.
  * *Why enable?* If you wish to run higher context lengths (e.g., 32k+ tokens), setting KV cache quantization to `Q8_0` or `Q4_0` reduces context RAM consumption by 50–75% with minimal quality loss.

---

### C. Inference & Sampling Settings (`Inference` Tab)

#### 16. Enable Thinking (Reasoning / Custom Fields)
* **Current Setting:** `Enabled (On)`
* **Suggested Setting:** `Enabled` (if using Qwen 3.5 Reasoning features) or `Disabled` for direct text responses.
* **Explanation & Reason:**
  * Toggles internal chain-of-thought output generation before providing final answers.

#### 17. Temperature
* **Current Setting:** `1.0`
* **Suggested Setting:** `0.6` to `0.7` for reasoning/coding; `0.8` for creative writing.
* **Explanation & Reason:**
  * `1.0` can produce inconsistent or rambling output on Qwen 3.5 models. A value of `0.6`–`0.7` strikes an optimal balance between factual accuracy, logical consistency, and natural language structure.

#### 18. Context Overflow
* **Current Setting:** `Truncate Middle`
* **Suggested Setting:** `Truncate Middle` or `Rolling Window` (Keep as is)
* **Explanation & Reason:**
  * Determines how LM Studio handles prompts that exceed the configured `Context Length`. `Truncate Middle` preserves system prompts and early conversation history along with recent user prompts.

#### 19. Top K Sampling
* **Current Setting:** `20`
* **Suggested Setting:** `40`
* **Explanation & Reason:**
  * Limits generation choices to the top *K* probable tokens. `20` is slightly narrow; `40` is standard for Qwen models to prevent repetitive vocabulary.

#### 20. Presence Penalty & Repeat Penalty
* **Current Setting:** `Presence Penalty: 1.5` | `Repeat Penalty: Disabled`
* **Suggested Setting:** `Presence Penalty: 0.0` to `0.2` | `Repeat Penalty: 1.05` to `1.1`
* **Explanation & Reason:**
  * A **Presence Penalty of 1.5 is excessively high** and can cause the model to introduce erratic topic shifts or strange synonym substitutions to avoid repeating words. 
  * Lower `Presence Penalty` to `0.0`–`0.2` and rely on a modest `Repeat Penalty` (`1.05`–`1.10`) for clean generation.

#### 21. Top P Sampling
* **Current Setting:** `0.95`
* **Suggested Setting:** `0.90` to `0.95` (Keep as is)
* **Explanation & Reason:**
  * Cumulative probability threshold for nucleus sampling. `0.95` works well across most Qwen 3.5 tasks.

---

## 3. Recommended Preset Summary Table

| Field Category | Setting Name | Screenshot Value | Recommended Value | Why Change? |
| :--- | :--- | :--- | :--- | :--- |
| **Load** | Context Length | 128,000 | **16,384 – 32,768** | Prevents Out-Of-Memory (OOM) crashes on 24 GB RAM. |
| **Load** | GPU Offload | 32 | **Max / All** | Maximizes Arc 140T iGPU execution. |
| **Advanced** | CPU Thread Pool Size | 6 | **6** | Matches physical P-cores on Intel Core Ultra 7. |
| **Advanced** | Eval Batch Size | 2048 | **512** | Smooths out RAM spikes during long prompt ingest. |
| **Advanced** | Offload KV Cache to GPU | Disabled | **Enabled** | Accelerates token generation using Arc iGPU memory. |
| **Advanced** | KV Cache Quantization | Disabled | **Q8_0 or Q4_0** *(Optional)* | Drastically cuts RAM usage if context > 16k. |
| **Sampling** | Temperature | 1.0 | **0.6 – 0.7** | Enhances logical precision and factual stability. |
| **Sampling** | Presence Penalty | 1.5 | **0.0 – 0.2** | Fixes potential topic drift and vocabulary distortion. |
| **Sampling** | Top K Sampling | 20 | **40** | Improves sentence naturalness and vocabulary range. |

---

## 4. Alternative Model Choice Tip (Q4_K_M vs Q8_0)

If you notice slow inference speeds (below 10–15 tokens/sec) or low free RAM, consider downloading **`Qwen3.5-9B-Q4_K_M.gguf`** (~5.5 GB disk size):
* **Memory savings:** Saves ~5 GB of system RAM compared to `Q8_0`.
* **Performance:** Up to 1.8x faster generation speed on shared-memory iGPUs due to reduced memory bandwidth demands.
* **Accuracy:** Retains over 98% of baseline intelligence.


Outline
Raw Markdown Mode
Table of Contents is not available in raw markdown view.

