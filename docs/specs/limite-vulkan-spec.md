# Specification: AMD Vulkan support for Limite 1B Violetto on RX 6900 XT

## 1. Findings, compatibility baseline and project objectives

### 1.1 Summary

Create a fork of `paradigma-inc/limite-violetto` that adds a dedicated Vulkan inference and serving implementation for the AMD Radeon RX 6900 XT.

The agreed compatibility boundary is:

- Preserve the model’s architecture, checkpoint, tokenizer, prompting rules and numerical behavior.
- Preserve the documented OpenAI-compatible serving workflow and mathematical evaluation capabilities.
- Preserve BF16 semantics, using software conversion and rounding where necessary.
- Qualify native Windows and Linux as equally supported platforms.
- Retain the existing CUDA/vLLM implementation.

The Vulkan implementation will use a pinned, patched **llama.cpp/GGML Vulkan backend**, extended with a native Limite architecture and model-specific numerical operations. It will expose the model through a separate serving package in the fork.

This is a substantial inference-engine integration. Changing a PyTorch device string, replacing the CUDA wheel index or adding a Vulkan platform detector to the existing plugin would not provide the required functionality.

“Full functionality” means the supported inference and serving capabilities of this model. It does not require implementing arbitrary PyTorch operations, preserving vLLM’s internal Python interfaces or adding training capabilities that the upstream model itself has not qualified.

### 1.2 Research status and immutable inputs

The analysis inspected:

- The GitHub serving implementation and configuration validation.
- The root and evaluator dependency manifests and lockfiles.
- The evaluator’s serving, request-generation and reproducibility contracts.
- The Hugging Face model card, native Transformers implementation, configuration, generation configuration and chat template.
- The Safetensors header, without downloading the complete weights.
- Relevant llama.cpp Vulkan code and merged patches.
- The local machine’s Vulkan device information.

Use these immutable inputs as the starting point:

| Input | Inspected revision |
|---|---|
| Limite GitHub repository | `7ddad34f5e3d25ddea68c6745fc00373b82bca21` |
| Violetto Hugging Face repository | `9f4d1567975a44c35127503deadfe52d434d734f` |
| Proposed llama.cpp dependency baseline | `296538adea19ff397100bf8e7d71f302d2d2bb35` |

The source and checkpoint snapshots are available in the [Limite repository](https://github.com/paradigma-inc/limite-violetto/tree/7ddad34f5e3d25ddea68c6745fc00373b82bca21) and [model repository](https://huggingface.co/paradigma-inc/limite-1b-violetto/tree/9f4d1567975a44c35127503deadfe52d434d734f).

The model contains:

- **1,035,253,888 parameters**.
- **1,035,141,120 BF16 parameters**.
- **112,768 FP32 parameters**.
- **743 serialized tensors**.
- A `model.safetensors` file of **2,070,811,520 bytes**, approximately **1.93 GiB**.

Record its published SHA-256:

```text
9dc7e89e30acf473629f98ac3ade535fb05cc044fe7e399b78d0bc957283a5f5
```

These are inspected metadata facts. No end-to-end Vulkan inference, performance measurement or numerical qualification has been performed during this analysis.

### 1.3 Verified dependency versions

There are **two different runtime environments** in the repository. They must remain distinct in the specification and implementation.

| Component | Root serving environment | Evaluator environment |
|---|---|---|
| Python requirement | `~=3.12.0` | `~=3.12.0` |
| PyTorch | `2.11.0+cu130` | `2.13.0+cu130` |
| vLLM | `0.26.0` | `0.28.0+cu129` |
| Transformers | `5.6.2` | `5.14.1` |
| Triton | `3.6.0` | `3.7.1` |
| FlashInfer Python | `0.6.14` | `0.6.16.post3` |
| NVIDIA CUDA runtime package | `13.0.96` | `13.0.96` |
| Tokenizers | `0.22.2` | `0.22.2` |

The root README’s advertised versions agree with the root lockfile. The evaluator has independently advanced its runtime and explicitly selects a CUDA 12.9 vLLM wheel alongside CUDA 13.0 PyTorch wheels. This combination is intentional in its manifest, but must not be assumed interchangeable with the root runtime. [Root lockfile](https://github.com/paradigma-inc/limite-violetto/blob/7ddad34f5e3d25ddea68c6745fc00373b82bca21/uv.lock), [evaluator manifest](https://github.com/paradigma-inc/limite-violetto/blob/7ddad34f5e3d25ddea68c6745fc00373b82bca21/limite-evals/pyproject.toml), [evaluator lockfile](https://github.com/paradigma-inc/limite-violetto/blob/7ddad34f5e3d25ddea68c6745fc00373b82bca21/limite-evals/uv.lock).

Release checks also establish:

- PyTorch **2.11.0** is a published release; its GitHub release was published on **23 March 2026**.
- Official PyTorch instructions provide **2.11.0 CUDA 13.0 and CPU wheels**.
- vLLM **0.26.0** is published; its GitHub release was published on **27 July 2026**.
- The inspected vLLM 0.26.0 package metadata requires `torch==2.11.0` and permits `transformers>=5.5.3`, making the repository’s Transformers 5.6.2 selection consistent with the declared requirement.
- Release-note mentions of a newer Transformers version do not override the wheel’s actual dependency metadata. [PyTorch release](https://github.com/pytorch/pytorch/releases/tag/v2.11.0), [official PyTorch wheel matrix](https://pytorch.org/get-started/previous-versions/#v2110), [vLLM release](https://github.com/vllm-project/vllm/releases/tag/v0.26.0), [vLLM CUDA requirements](https://github.com/vllm-project/vllm/blob/v0.26.0/requirements/cuda.txt).

CUDA **13.0** is a runtime/toolkit family, rather than an AMD-compatible execution interface. NVIDIA documents driver branch **580 or later** for CUDA 13.x minor-version compatibility. CUDA support-library version numbers are individually assigned and must not be interpreted as the toolkit version. [CUDA 13.0 release notes](https://docs.nvidia.com/cuda/archive/13.0.0/cuda-toolkit-release-notes/index.html).

**Dependency policy for the fork:**

1. Keep the existing root CUDA lockfile intact.
2. Keep the evaluator’s existing CUDA environment available for historical reproducibility.
3. Give Vulkan its own package manifest and lockfile.
4. Use PyTorch **2.11.0 CPU**, Transformers **5.6.2** and Tokenizers **0.22.2** for conversion and the portable reference tooling.
5. Use the original **2.11.0+cu130 / vLLM 0.26.0** environment as the primary GPU numerical reference.
6. Record the evaluator’s newer environment separately when it is used as a comparison engine.
7. Require hardware qualification before promoting dependency updates.

The Vulkan server must not import vLLM or require CUDA, ROCm, Triton or FlashInfer.

### 1.4 Actual target GPU capabilities

The local Windows device inspection reported:

| Property | Observed value |
|---|---|
| Device | AMD Radeon RX 6900 XT |
| Vendor/device IDs | `0x1002 / 0x73bf` |
| Device Vulkan API | `1.4.315` |
| Vulkan instance version | `1.4.309` |
| Driver information | `25.10.30.02` |
| Maximum individual memory allocation | `0x80000000`, or 2 GiB |
| Maximum buffer size | 2 GiB |
| Maximum storage-buffer range | `4,294,967,295` bytes |
| Maximum compute shared memory | 32 KiB |
| Reported subgroup size | 64 |
| Supported subgroup-size range | 32–64 |
| FP16 shaders | Supported |
| 16-bit storage-buffer access | Supported |
| Cooperative-matrix feature | Reported supported |

These observations replace the earlier ambiguous “Vulkan 2.56.0” version description for this machine.

A supported cooperative-matrix feature does not establish which operand types, accumulator types or matrix dimensions are available. The implementation must query those properties separately.

The runtime must work without native BF16 arithmetic or BF16 cooperative matrices. Querying Vulkan version alone is insufficient for selecting an execution path.

### 1.5 Relevant public implementations and patches

Use the following projects as engineering references:

| Project or patch | Relevant contribution | Application to this fork |
|---|---|---|
| llama.cpp Vulkan backend | Established compute backend, shader generation, memory management and attention | Primary backend foundation |
| llama.cpp PR **#23420** | BF16 K/V flash-attention support, including an FP32 fallback | BF16 cache handling and portable attention |
| llama.cpp PR **#26675** | Explicit accumulator and source-precision controls | Prevent unintended FP16 or quantized intermediate paths |
| llama.cpp PR **#26578** | Vulkan operations for architecture-specific hyper-connections | Pattern for adding custom Limite graph operations |
| llama.cpp PR **#15352** | BF16 shader optimizer workaround | Shader compiler compatibility testing |
| `mudler/vllm.cpp` | Vulkan platform/backend experiments | Capability reporting and fallback accounting |
| `defilantech/llmkube-runtimes` | Hardware-gated Vulkan runtime builds | Packaging and detection of silent CPU fallback |

PR #23420 merged on **30 May 2026**. Its description explicitly distinguishes BF16 cache storage from general BF16 arithmetic and provides an FP32 fallback. [BF16 Vulkan attention patch](https://github.com/ggml-org/llama.cpp/pull/23420).

PR #26675 merged on **8 September 2026**; its changes are also described in the [GGML 0.24.0 release](https://github.com/ggml-org/ggml/releases/tag/v0.24.0).

The hyper-connection and shader-optimizer patches are merged examples of extending Vulkan functionality and fixing compiler-specific failures. Their mathematical operations must not be substituted for Limite’s operations. [Custom connection operations](https://github.com/ggml-org/llama.cpp/pull/26578), [BF16 optimizer fix](https://github.com/ggml-org/llama.cpp/pull/15352).

The inspected `vllm.cpp` inventory describes its Vulkan implementation as incomplete and explicitly says it has not demonstrated complete model execution. It is useful inspiration, but is not a ready replacement for this model’s runtime. [Vulkan backend inventory](https://github.com/mudler/vllm.cpp/blob/main/.agents/backend-matrix.md).

The inspected llama.cpp architecture and conversion registries did not contain Limite. The work therefore requires an actual architecture implementation, not simply checkpoint conversion. [Conversion registry](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/conversion/__init__.py).

---

## 2. Fork architecture, packaging and execution design

### 2.1 Repository organization

Keep the original Python plugin as the CUDA implementation.

Add three components:

1. **A separately installable Vulkan package**
   - Own Python 3.12 environment and lockfile.
   - CLI, checkpoint preparation, canonical prompt rendering and public API.
   - No dependency on the CUDA plugin during normal Vulkan execution.

2. **A pinned native dependency with a maintained patch series**
   - llama.cpp/GGML baseline fixed to the inspected SHA.
   - Limite architecture registration, tensor loading, graph construction and Vulkan operations.
   - Native worker derived from llama.cpp’s serving infrastructure.
   - Patches applied reproducibly, with verification that the expected source revision is present.

3. **Backend-aware evaluator integration**
   - Preserve existing scoring and dataset contracts.
   - Add Vulkan launch and fingerprint adapters.
   - Separate scoring dependencies from CUDA-engine dependencies.

Use an additive subproject for Vulkan rather than converting the root package into a mixed CUDA/Vulkan dependency graph. The existing `uv sync --locked` command will continue to install the original CUDA environment.

Document the Vulkan installation separately and make its command independent of the user’s current working directory.

### 2.2 Proposed public CLI

Introduce:

```text
limite-vulkan doctor
limite-vulkan prepare
limite-vulkan inspect
limite-vulkan serve
limite-vulkan complete
```

Their responsibilities are:

| Command | Required behavior |
|---|---|
| `doctor` | Enumerate devices, query capabilities and limits, run a small compute check, report selected backend |
| `prepare` | Resolve the pinned checkpoint, verify hashes and create the lossless runtime artifact |
| `inspect` | Display model identity, conversion metadata, numerical profile and estimated memory requirements |
| `serve` | Launch the native worker and expose the OpenAI-compatible API |
| `complete` | Run local prompt or chat inference through the same execution and sampling implementation |

Example target interface:

```powershell
limite-vulkan prepare `
  --model paradigma-inc/limite-1b-violetto `
  --revision 9f4d1567975a44c35127503deadfe52d434d734f

limite-vulkan serve `
  --model paradigma-inc/limite-1b-violetto `
  --device auto `
  --precision bf16-semantics `
  --max-model-len 8192 `
  --max-num-seqs 4
```

These are proposed commands to implement, not commands supported by the current repository.

Defaults:

- Bind public serving to `127.0.0.1:8000`.
- Select the RX 6900 XT automatically when it is the only suitable discrete GPU.
- Require explicit selection when multiple equally suitable devices exist.
- Use `bf16-semantics`.
- Default serving context: **8,192 tokens**.
- Allow explicit context configuration up to **131,072 tokens**.
- Default active sequences: **4**, subject to memory admission.
- Initial prefill chunk: **512 tokens**.
- Queue requests that temporarily exceed active capacity.
- Refuse startup if a complete Vulkan execution graph cannot be supported.

Automatic selection must not silently choose a software Vulkan implementation for a supported-GPU serving run.

### 2.3 Native worker and API facade

Use a Python API facade and a native Vulkan worker.

The facade owns:

- Public request validation.
- The pinned Hugging Face tokenizer.
- Canonical Jinja chat rendering.
- Resolving generation defaults.
- Mapping native events to OpenAI responses.
- Worker lifecycle and readiness checks.
- Request cancellation and structured error reporting.

The native worker owns:

- GGUF loading and model graph execution.
- GPU weights and K/V cache.
- Continuous batching and chunked prefill.
- Sampling and stop handling.
- Numerical-profile enforcement.
- Backend coverage and GPU timing telemetry.

Reuse llama.cpp’s serving infrastructure, but add a private Limite interface that accepts **already tokenized prompts**. Do not send a rendered string to a second tokenizer and assume equivalent token IDs.

The private worker listens on an automatically assigned loopback port. Protect it with a per-launch token and verify its launch identity before accepting it as the worker for this server instance.

Required private operations:

```text
GET  /limite/v1/fingerprint
POST /limite/v1/generate
POST /limite/v1/cancel
```

A generation request carries:

- Request and choice identifiers.
- Prompt token IDs.
- Resolved sampling parameters.
- Maximum output length.
- Stop strings and stop-token IDs.
- Requested log-probability information.
- Detokenization policy.

Streaming worker events carry:

- Token IDs and choice index.
- Stable output text fragments.
- Requested token log probabilities.
- Completion status and stop reason.
- Usage counts.

Use a versioned private protocol. Reject a facade/worker protocol mismatch at startup.

### 2.4 Checkpoint preparation and GGUF conversion

Implement a Limite-specific converter against the pinned llama.cpp conversion framework.

The default converted artifact must be **lossless with respect to the serialized checkpoint**:

- Retain BF16 matrices as BF16 data.
- Retain FP32 scalars, gates and mixer parameters as FP32.
- Preserve tensor orientation and logical shape.
- Preserve all architecture fields and numerical-contract fields.
- Preserve the original embedding and output-head tie.
- Preserve tokenizer assets, special-token definitions and chat-template identity.
- Store upstream checkpoint revision and source hashes.

Do not convert the model to FP16 or an integer quantization by default.

For the pinned Violetto snapshot:

- Require exactly **743 source tensors**.
- Validate each tensor’s name, dtype and shape.
- Reject missing, unexpected or duplicate source mappings.
- Include a complete tensor-map manifest.
- Store projection scales as original parameters.
- Apply projection-scale folding during runtime loading.
- Mark derived folded buffers explicitly so reloading cannot apply a scale twice.

Conversion may fuse Q/K/V and gate/up matrices by concatenation. The mapping must remain reversible and auditable.

Important matrix shapes:

| Tensor family | Hugging Face logical shape |
|---|---|
| Token embedding / tied head | `[151680, 1280]` |
| Value embedding | `[151680, 256]` |
| Q projection | `[1280, 1280]` |
| K projection | `[256, 1280]` |
| V projection | `[256, 1280]` |
| Fused QKV | `[1792, 1280]` |
| O projection | `[1280, 1280]` |
| MLP gate/up projections | Each `[3328, 1280]` |
| Fused gate/up | `[6656, 1280]` |
| MLP down projection | `[1280, 3328]` |
| MUDD dense1 | `[32, 1280]` |
| MUDD dense2, each branch | `[48, 3, 32]` |
| MUDD bias, each branch | `[48, 3]` |

The converter must not replace Limite’s architecture identifier with Llama, Qwen or another approximately similar architecture.

An existing community GGUF may be inspected as a comparison artifact. It must not replace the pinned official checkpoint as the qualification input.

### 2.5 Build and dependency controls

Initial build targets:

- **Windows 11 x86-64**, native execution.
- **Ubuntu 24.04 LTS x86-64**, native Vulkan execution.

Use:

- C++17 or the higher language standard required by the pinned dependency.
- CMake **3.31.x** as the initial build-tool baseline.
- MSVC v143 on Windows.
- GCC 13 on Ubuntu.
- LunarG Vulkan SDK **1.4.328.1** as the initial shader-toolchain candidate.
- SPIR-V targeting **Vulkan 1.3** for the portable execution path.

The SDK selection is a proposed build baseline, not a claim that the complete patched engine already builds with it. A successful shader and hardware qualification is mandatory.

The runtime must not require the Vulkan SDK to be installed on the end user’s machine. Release binaries include their compiled shaders and require only a compatible driver/loader.

Record exact compiler patch versions, SDK build, `glslc` version and shader hashes in release metadata.

Build configuration must explicitly disable CUDA and HIP for Vulkan artifacts. Do not allow a locally installed CUDA toolkit to affect the selected backend.

Shader capability tests must check:

- Compiler exit status.
- Successful SPIR-V generation.
- SPIR-V validation.
- Actual device support before enabling a runtime variant.

A compiler accepting an extension declaration is not sufficient proof of usable device functionality.

### 2.6 GPU execution coverage

The supported Vulkan profile requires GPU execution of:

- Embedding lookup.
- All projections and MLP matrix operations.
- Normalization.
- RoPE.
- Value-embedding operations.
- Attention and cache updates.
- XSA.
- Residual joins and MUDD mixing.
- Output projection and logit softcapping.

CPU execution is permitted for tokenization, request processing, scheduling decisions, sampling and output formatting.

No transformer-layer operation may silently execute on the CPU.

Implement a startup coverage check against the actual loaded graph. If any required node lacks a Vulkan implementation, report the operation and fail startup.

---

## 3. Model architecture and numerical requirements

### 3.1 Architecture contract

Implement the configuration actually published with Violetto:

| Property | Value |
|---|---|
| Decoder layers | 48 |
| Hidden size | 1,280 |
| Query heads | 10 |
| K/V heads | 2 |
| Head dimension | 128 |
| GQA ratio | 5 query heads per K/V head |
| MLP | SwiGLU |
| MLP intermediate size | 3,328 |
| Maximum context | 131,072 |
| Attention scale | `0.1` |
| Global layers, zero-based | `3, 7, 11, …, 47` |
| Local layers | Remaining 36 layers |
| Serialized local window | 1,024 |
| Effective local attention span | 1,025 keys |
| Global attention window | Unlimited within configured context |
| Global RoPE | Disabled |
| Local rotary dimensions | First 64 dimensions of each head |
| Local rotary pairs | 32 |
| Rotary base | 1,024 |
| Value-embedding layers | `1, 4, 7, …, 46` |
| XSA | All 48 layers |
| MUDD layers | 24 and 47 |
| MUDD taps | Three |
| MUDD residual branch | Enabled |
| Tied embedding/head | Enabled |
| Head precision mode | `oracle_exact` |
| Final hidden-state softcap | Disabled |

These values come from the [pinned model configuration](https://huggingface.co/paradigma-inc/limite-1b-violetto/blob/9f4d1567975a44c35127503deadfe52d434d734f/config.json).

Architecture validation must run before graph construction or GPU allocation. Unsupported configurations must produce specific errors rather than entering a generic fallback graph.

The first qualification target is the pinned Violetto checkpoint. Other Limite checkpoints may be admitted only when their configuration uses implemented operations and passes separate qualification.

### 3.2 BF16 execution profile

The supported profile is named:

```text
bf16-semantics
```

It means:

- Original BF16 checkpoint values remain exact.
- BF16-valued intermediate tensors are rounded at the same operation boundaries as the reference implementation.
- FP32 parameters remain FP32 until the reference explicitly narrows them.
- FP32 exceptions, including XSA calculations and final softcapping, remain FP32.
- Reduction implementations may differ across hardware, but must pass the numerical and quality gates.

It does **not** imply bitwise identity with CUDA’s GEMM or attention kernels.

A portable implementation can represent a BF16-valued activation using FP32 storage whose values are restricted to the BF16 representable set. This is valid only when it preserves the required rounding boundaries.

Implement reusable Vulkan primitives for:

- Packed BF16 load/store.
- FP32-to-BF16 round-to-nearest-even.
- BF16-to-FP32 exact expansion.
- BF16-rounded multiply.
- BF16-rounded add.
- Operation-specific FP32 reductions.

Handle signed zero, infinities and NaNs explicitly. Do not use simple truncation for FP32-to-BF16 conversion.

Prevent compiler contraction or graph fusion across required rounding points. In particular:

```text
round_bf16(round_bf16(a * b) + round_bf16(c * d))
```

must not become an unrestricted FP32 fused multiply-add.

Use GGML precision controls where supported, and add model-specific checks where the backend does not enforce them sufficiently. The inspected GGML API provides accumulator and source-precision controls; neither an API call nor a build flag substitutes for numerical testing. [Precision-control changes](https://github.com/ggml-org/ggml/releases/tag/v0.24.0).

### 3.3 Normalization

Limite uses gain-free RMS normalization with the default epsilon derived from the tensor dtype.

For BF16-valued tensors:

```text
epsilon = finfo(bfloat16).eps = 0.0078125
```

Do not substitute a common fixed epsilon such as `1e-6`.

Apply normalization to:

- The initial token embedding.
- Attention inputs.
- Q and K, independently per head.
- MLP inputs.
- The final hidden state.

Implement FP32 accumulation for the normalization reduction and BF16 rounding for the output, matching the qualified reference behavior.

Test zero vectors, very small vectors, high-magnitude vectors and all supported row widths.

### 3.4 Projection-scale folding

Each attention layer has learned `qkv_scale` and `o_scale` parameters.

The reference’s scalar handling narrows the scale to the matrix dtype before forming the BF16-scaled projection weights.

The effective fold is:

```text
folded_weight =
    round_bf16(
        round_bf16(scale) * original_bf16_weight
    )
```

Preserve the original FP32 scalar in the artifact.

Fold once after a complete raw-weight load. Keep raw and derived state distinguishable.

Reloading must reconstruct the same effective weights and must not multiply already folded weights again. The upstream implementation already tests this failure mode. [Weight-loading implementation](https://github.com/paradigma-inc/limite-violetto/blob/7ddad34f5e3d25ddea68c6745fc00373b82bca21/src/limite_vllm/modeling.py).

### 3.5 Partial RoPE

Implement Limite’s exact rotary layout:

- Adjacent interleaved pairs.
- Only 32 pairs have nonzero frequencies.
- Remaining dimensions pass through unchanged.
- Odd sine lanes use the negative sign.
- Cosine and sine values are BF16 before applying the rotation.
- Global layers bypass rotary entirely.

The frequency rule is:

```text
frequency[j] = (1 / 1024) ** linspace(0, 1, 32)[j]
```

This must not be replaced with a conventional rotary-frequency formula.

For each active pair:

```text
(a, b) -> (a*cos + b*sin, b*cos - a*sin)
```

Preserve BF16 multiplication and addition boundaries.

Use absolute sequence positions. Cache eviction and chunk boundaries must not reset the position index.

To reduce cross-platform transcendental differences, ship a versioned reference-derived BF16 cosine/sine table for the pinned architecture:

- Positions `0` through `131071`.
- 32 active pairs.
- Both cosine and sine.
- Approximately **16 MiB** before packaging overhead.

Generate and fingerprint this table using the pinned reference implementation. Include the frequency-construction metadata and generator version.

The table removes a driver-specific source of numerical variation; it does not make GEMM or attention bitwise identical to CUDA.

### 3.6 Value embeddings

Violetto uses a shared value-embedding table with two stored heads.

At each configured value-embedding layer:

1. Look up the current token’s value embedding.
2. Interpret it as `[2, 128]`.
3. Compute the gate from the first 12 attention-input channels.
4. Apply the learned FP32 gate matrix narrowed as the reference requires.
5. Apply sigmoid and scale by `2.0`.
6. Add the gated embedding to projected V.

The **modified V** must be:

- Consumed by attention.
- Written into the K/V cache.
- Retained as the current value vector for XSA.

Caching the unmodified V projection would change future-token behavior.

### 3.7 Attention

Implement grouped-query causal attention with:

```text
query_heads = 10
kv_heads = 2
scale = 0.1
```

Do not use `1/sqrt(128)` as the scale.

Local masks admit:

```text
q - 1024 <= k <= q
```

This includes **1,025** positions when sufficient history exists.

Global masks admit:

```text
0 <= k <= q
```

For cached/chunked prefill, masks must use absolute query and key positions. A non-square query/key matrix must not be handled with an assumed square causal mask.

Use tiled attention with online softmax and bounded workspace. Do not allocate a full context-by-context attention matrix.

FP32 attention accumulation is the portable baseline. Round attention outputs to BF16 at the reference boundary.

Qualify the tiled implementation against the selected CUDA attention reference. The native eager attention implementation is useful for small-tensor diagnostics, but its intermediate rounding is not automatically identical to fused SDPA or FlashAttention.

Log the actual CUDA backend selected for reference runs.

### 3.8 XSA

XSA uses the current token’s value vector, expanded logically across GQA groups.

For each query head:

```text
vn = normalize(current_value.float(), epsilon=1e-4)

projection = sum(attention_output.float() * vn)

alpha = tanh(xsa_alpha.float())

correction = alpha * projection * vn

output = attention_output - correction.to(output_dtype)
```

Requirements:

- Normalize with a denominator clamped to `1e-4`.
- Keep normalization, projection and correction arithmetic in FP32.
- Narrow the correction before the BF16 subtraction.
- Use the current post-value-embedding V, not an attention-weighted value vector.
- Map each group of five query heads to its associated K/V head.
- Preserve all ten per-head `xsa_alpha` values.

Precompute `tanh(xsa_alpha)` from the FP32 parameters during loading, using qualified reference values.

### 3.9 Attention gates and residual joins

The attention output gate uses:

- The first 128 attention-input channels.
- A `[10, 128]` learned gate matrix.
- Sigmoid.
- Scale `2.0`.
- Application per head before the O projection.

Each layer’s residual joins use learned coefficients:

```text
attention_join =
    resid_lambda_attn * residual_base
    + post_lambda_attn * attention_branch

layer_output =
    resid_lambda_mlp * attention_join
    + post_lambda_mlp * mlp_branch
```

These are not ordinary additive residuals.

Narrow coefficients and round products and sums according to the BF16 contract.

At MUDD layers, `residual_base` is the R-way mixer output.

### 3.10 MUDD mixing

Implement both the attention-input mixer and the separately parameterized residual-base mixer.

History convention:

```text
hist[0] = normalized token embedding
hist[j] = output of layer j - 1
```

Tap sets:

```text
layer 24: hist[0], hist[12], hist[24]
layer 47: hist[0], hist[23], hist[47]
```

For each token:

1. Normalize the current residual stream.
2. Project through `dense1`.
3. Apply GELU using the reference’s default, erf-based formulation.
4. Project through the absolute-layer slice of `dense2`.
5. Add the matching bias.
6. Multiply each tap by its coefficient.
7. Accumulate the taps **left to right**, rounding at the BF16 boundaries.

The R-way shares `dense1` and the tap values, but uses `dense2_mlp` and `bias_mlp`.

Do not:

- Replace GELU with a tanh approximation.
- Normalize mixer coefficients with softmax.
- Collapse ordered accumulation into a differently associated reduction.
- Treat layer-history taps as temporal sequence history.
- Retain every layer activation unnecessarily.

Only keep residual snapshots needed by future tap sets for the current batch/chunk.

### 3.11 SwiGLU and nonlinear operations

Violetto’s MLP is:

```text
down_proj(silu(gate_proj(h)) * up_proj(h))
```

There is no additional clamp.

Preserve BF16 rounding after projection, SiLU and multiplication.

For BF16-input nonlinear functions, use reference-derived lookup tables where this avoids shader-library approximation differences:

- GELU with `approximate="none"`.
- SiLU.
- Sigmoid.

Each table has at most 65,536 BF16 input patterns and is small relative to the model.

Generate tables under the pinned numerical reference, verify special-value behavior and include hashes. The shader must index using the exact BF16 representation.

This is a correctness mechanism, not a substitute for testing surrounding arithmetic.

### 3.12 Output head and softcapping

The output head is tied to the token embedding.

For `oracle_exact`:

1. Compute the output projection.
2. Round the raw projection result to BF16.
3. Expand that result to FP32.
4. Apply:

```text
logits = 23 * sigmoid((raw_logits + 5) / 7.5)
```

Do not substitute the tanh softcap used by other model families.

A lookup table indexed by raw BF16 logits may implement this complete FP32 softcap expression for the pinned constants. Its output remains FP32.

Preserve:

- Matrix width **151,680**.
- Tokenizer vocabulary size **151,667**.
- The distinction between output rows and tokenizer-defined IDs.
- The exact upstream sampling behavior for padded rows.

Do not silently delete the extra 13 rows, renormalize a narrower distribution or add an unverified padded-token mask.

Represent unused tokenizer rows explicitly in the converted metadata. Tests must compare their logits and detokenization behavior with the reference.

### 3.13 Memory and K/V cache

Use BF16 K/V storage.

The idealized cache requirement per stored token per layer is:

```text
2 tensors × 2 heads × 128 dimensions × 2 bytes
= 1024 bytes
```

At full context:

| Cache component | Idealized size |
|---|---|
| Twelve global layers | **1.5 GiB** |
| Thirty-six local layers retaining 1,025 tokens | **36.04 MiB** |
| Total persistent K/V for one full-length sequence | **Approximately 1.535 GiB** |

This excludes page alignment, temporary prefill storage, allocator overhead and workspace.

Use:

- Paged, append-only global caches.
- Bounded local caches.
- **64-token pages** as the initial page size.
- Temporary local prefill storage so writing a chunk cannot overwrite keys still needed by earlier queries in that chunk.
- Absolute position metadata for every cache segment.
- Per-sequence ownership and deterministic reclamation.

With 64-token page alignment, a local retention span requires 17 pages, or 1,088 token slots per local layer.

Do not allocate a full-length cache for every local layer.

Respect the observed 2 GiB allocation/buffer limit:

- Use segmented allocations.
- Cap ordinary allocation slabs at **512 MiB**, reduced further if device limits require it.
- Do not combine the complete cache into one oversized Vulkan buffer.
- Check storage-buffer ranges and address arithmetic.
- Exercise segmentation with artificially smaller limits during testing.

Default GPU memory budget:

```text
min(12 GiB, reported available device-local budget - 1 GiB)
```

Include weights, derived weights, cache, workspace and staging in admission accounting.

When memory-budget reporting is unavailable, use a conservative physical-heap estimate and label it as estimated.

Reserve memory before admitting a request. Capacity failures must not surface halfway through generation when they could have been detected at admission.

### 3.14 Dispatch and synchronization

Start with a compute-queue implementation using explicit dependencies between:

- Uploads.
- Cache writes.
- Attention reads.
- Layer operations.
- Output reads.
- Cancellation and buffer reclamation.

Reuse command buffers and staging allocations after correctness is established.

Do not assume fixed subgroup width. Specialize for supported widths and retain a tested portable path.

Limit shared-memory use to queried device capabilities; the observed Windows limit is 32 KiB.

Avoid long uninterrupted dispatches on the Windows display GPU. Tile global attention over key ranges and split large prefill work. Use a **250 ms maximum measured dispatch duration** as the initial qualification limit, reducing tile sizes when necessary.

The implementation must not require disabling Windows’ watchdog.

---

## 4. Serving behavior, tokenizer fidelity and evaluator integration

### 4.1 Supported public endpoints

Provide:

```text
GET  /health
GET  /version
GET  /v1/models
POST /v1/chat/completions
POST /v1/completions
GET  /limite/runtime
GET  /metrics
```

Required generation capabilities:

- Non-streaming and SSE streaming.
- Greedy and sampled decoding.
- Temperature, top-p, top-k and min-p.
- Seeded generation.
- Presence, frequency and repetition penalties.
- Output-length limits.
- Stop strings and stop-token IDs.
- Multiple choices through `n`.
- Token usage accounting.
- Generated-token log probabilities and top log probabilities.
- Prompt log probabilities for completion requests.
- Request cancellation.
- Concurrent requests and continuous batching.
- Raw mathematical reasoning output.

Support string prompts and token-ID prompts on `/v1/completions`.

Fields outside the declared compatibility surface must produce an explicit unsupported-parameter error when supplied with behavior-changing values. Do not accept and ignore them.

This does not promise every vLLM option. Publish the supported parameter matrix.

### 4.2 Generation defaults and sampling

Load checkpoint generation defaults:

```text
temperature = 0.6
top_p = 0.95
do_sample = true
eos_token_id = 151643
pad_token_id = 151643
```

Explicit request parameters override defaults.

Use the pinned vLLM 0.26.0 behavior as the parameter-semantics reference:

- Copy penalty and filtering order from the inspected implementation.
- Test the resulting distributions against fixed synthetic logits.
- Distinguish logits used for reporting log probabilities from probabilities modified for sampling.
- Define temperature zero as greedy decoding.
- Apply EOS and explicit stop-token behavior consistently.

Use a per-choice Philox counter-based RNG so a request’s random stream is independent of scheduler interleaving and batch membership.

Repeated requests with the same prompt, explicit seed and parameters must reproduce within a qualified platform/build.

Identical seeds across CUDA and Vulkan do not imply identical generated text: the sampling implementations and logits may differ slightly. Qualification compares numerical behavior and quality, rather than promising identical stochastic trajectories.

### 4.3 Canonical prompting

Use the checkpoint’s exact chat template.

The canonical system message is:

```text
You are a helpful assistant.
Please reason step by step, and put your final answer within \boxed{}.
```

Preserve the template’s:

- Initial `<|endoftext|>`.
- ChatML role markers.
- Whitespace.
- Generation prefix.
- Handling of assistant reasoning content.
- Validation of system messages and message roles.

Reject tools, tool calls, multimodal content and unsupported roles exactly where the canonical template rejects them.

Recommend a fresh single-turn problem for intended mathematical use. Do not introduce a stricter single-turn validator than the published template: it can serialize user and assistant messages even though the model card recommends avoiding prior history. [Pinned chat template](https://huggingface.co/paradigma-inc/limite-1b-violetto/blob/9f4d1567975a44c35127503deadfe52d434d734f/chat_template.jinja).

### 4.4 Tokenizer and text-output fidelity

The tokenizer identifies itself as `Qwen2Tokenizer`. Use its published assets as the authoritative tokenization reference.

Verify:

- Ordinary text.
- Mathematical notation and LaTeX.
- Unicode symbols.
- Whitespace and newline boundaries.
- Byte-fallback behavior.
- Added tokens.
- ChatML markers.
- Reasoning delimiters.
- Empty strings.
- Invalid or incomplete UTF-8 sequences.

Important IDs:

```text
151643 = <|endoftext|>
151644 = <|im_start|>
151645 = <|im_end|>
151665 = <think>
151666 = </think>
```

The reasoning delimiters are added tokens marked **non-special**. They must survive `skip_special_tokens=true`.

Do not install a reasoning parser that moves the `<think>` content into another response field. The evaluator expects raw text in `message.content`.

Streaming must:

- Emit only stable decoded text.
- Buffer incomplete UTF-8 sequences.
- Detect stop strings across token and event boundaries.
- Exclude terminating stop text when requested.
- Produce the same final text as non-streaming generation.
- Preserve usage counts for generated tokens even when terminators are omitted from returned text.

### 4.5 Continuous batching and context management

Use native worker scheduling with:

- Separate queued, prefill, decode, cancelled and completed states.
- Decode scheduling between bounded prefill chunks.
- Per-request generation state and RNG.
- K/V page allocation independent of request order.
- Admission based on requested context/output budget.
- Guaranteed cleanup after completion or cancellation.

A full-context deployment must support a request whose:

```text
prompt_tokens + maximum_generated_tokens <= 131072
```

Requests exceeding the configured serving context must fail before generation. Do not silently truncate prompts or shift the context.

Disabling context shifting is mandatory for qualified mathematical inference.

Prefix caching may be enabled only after parity tests pass. Cache identities must include:

- Token IDs.
- Checkpoint revision.
- Numerical profile.
- Graph/conversion version.
- Rendering identity where applicable.

A valid prefix hit must include both global and local-layer state. Reusing only global cache pages while losing the local attention tail is incorrect.

### 4.6 Errors and operational reporting

Required startup errors include:

- No usable Vulkan device.
- Unsupported graph operation.
- Unsupported checkpoint configuration.
- Shader/device capability mismatch.
- Artifact checksum failure.
- Incompatible facade/worker protocol.
- Insufficient memory for the requested configuration.
- Port already occupied.

Required request errors include:

- Unknown model.
- Invalid token ID.
- Invalid sampling parameter.
- Unsupported feature.
- Context overflow.
- Queue saturation.
- Worker failure.
- Device loss.

Return structured OpenAI-style errors with a clear message and parameter where applicable.

For device loss:

- Fail affected requests.
- Clear readiness.
- Preserve diagnostics.
- Require a fresh worker instance before serving resumes.
- Do not fall back to CPU and continue returning unqualified results.

The runtime fingerprint must include:

- Actual model revision and artifact hash.
- Tokenizer/template hashes.
- Numerical profile.
- Patched llama.cpp revision.
- Patch-series and shader hashes.
- Platform and driver identity.
- Device UUID.
- Context and batching configuration.
- Backend coverage.
- Launch identity.
- Precision and cache-storage choices.

Metrics must include queue depth, active sequences, prefill/decode throughput, cache occupancy, allocation totals, request failures and CPU fallback counts.

### 4.7 Evaluator changes

Add an engine abstraction to the existing evaluator:

```text
engine = cuda-vllm | vulkan
```

Default to the existing CUDA behavior.

For each engine, provide:

- Launch command construction.
- Readiness checks.
- Version/build verification.
- Actual served-model verification.
- Rendering identity verification.
- Runtime fingerprint collection.
- Cleanup.

Do not spoof a vLLM version from the Vulkan worker to pass the current evaluator gate.

Extend manifests with:

```text
engine_kind
engine_version
engine_build
numerical_profile
qualification_id
source_checkpoint_revision
runtime_artifact_hash
```

Retain the existing model, rendering, dataset, scorer, sampling and generation-budget fields.

For cross-engine comparisons:

- Require an explicit comparison mode permitting engine differences.
- Continue requiring matching measurement semantics.
- Show engine and numerical-environment differences in the report.
- Preserve historical CUDA manifests without rewriting them.
- Do not label Vulkan runs as CUDA-identical.

The evaluator currently refuses parsed reasoning output and carries `stop_reason`, `finish_reason` and completion-token counts through its request path. The Vulkan adapter must preserve those behaviors. [Evaluator request implementation](https://github.com/paradigma-inc/limite-violetto/blob/7ddad34f5e3d25ddea68c6745fc00373b82bca21/limite-evals/src/limite_evals/runner.py).

Keep conversion outside evaluation. Evaluation consumes a previously prepared artifact and records its relationship to the official checkpoint.

Separate the evaluator’s scoring environment from its CUDA engine packages so native Windows and Linux Vulkan evaluation can run without installing NVIDIA dependencies.

Before modifying the evaluator, follow its repository instructions and preserve immutable per-run artifacts. [Evaluator development contract](https://github.com/paradigma-inc/limite-violetto/blob/7ddad34f5e3d25ddea68c6745fc00373b82bca21/limite-evals/docs/development.md).

---

## 5. Verification, milestones and release acceptance

### 5.1 Reference artifacts

Generate a qualification bundle using the pinned CUDA reference environment.

Include:

- Exact dependency and hardware fingerprints.
- Source/checkpoint hashes.
- Tokenization and rendered-prompt fixtures.
- Intermediate tensors at selected layer boundaries.
- Prefill and cached-decode logits.
- Reference-derived nonlinear and RoPE tables.
- Small attention and mixer fixtures.
- Commands and environment settings.
- Selected attention backend.
- Machine-readable comparison results.

Use the native Transformers implementation and CUDA vLLM implementation as complementary references.

The CPU implementation is useful for structural checks and debugging. It cannot replace GPU qualification against the published runtime.

No official support claim may be made merely because the graph executes or the generated text looks plausible.

### 5.2 Conversion tests

Require:

- Complete 743-tensor inventory validation.
- BF16 and FP32 data preservation.
- Correct dimensions and matrix orientation.
- Correct fused-projection slices.
- Correct tied-head handling.
- Lossless round-trip reconstruction of source tensors.
- Missing/unexpected/duplicate tensor rejection.
- Corrupt and truncated artifact rejection.
- Stable conversion metadata.
- Repeated-load scale-folding correctness.
- Preservation of tokenizer and template identities.

Changing converter behavior must change the conversion version and derived artifact identity.

### 5.3 Primitive and architecture tests

Test Vulkan implementations against independent reference calculations.

Mandatory cases:

| Area | Test cases |
|---|---|
| BF16 conversion | Rounding ties, signed zero, infinities, NaNs, boundary values |
| RMSNorm | BF16 epsilon, zero rows, extreme magnitudes, head/hidden widths |
| RoPE | First position, large positions, untouched dimensions, odd-lane sign |
| Attention | Scale `0.1`, GQA mapping, causal masking, mixed global/local layers |
| Local window | Positions around 1023, 1024 and 1025; exactly inclusive boundary |
| Cached attention | Unequal query/key lengths, multi-token cached prefill |
| Value embeddings | Gate channels, designated layers, modified V written to cache |
| XSA | Near-zero V, current V selection, FP32 correction, all head mappings |
| MUDD | Both branches, absolute layer indexing, tap history, ordered accumulation |
| SwiGLU | Projection split, SiLU, product rounding, down projection |
| Residual joins | Coefficient narrowing, separate product/add rounding |
| Output head | Tie, BF16 raw projection, FP32 sigmoid softcap, padded rows |

Operations with exact integer/conversion behavior require bitwise agreement.

GEMM, attention and reduction operations use explicit numerical tolerances. Failures must identify the first divergent operation or layer.

### 5.4 Prefill, decode and batching parity

Test the same token sequence through:

1. One complete prefill.
2. Multiple prefill chunks.
3. Prefill followed by single-token cached decode.
4. Different legal chunk sizes.
5. A batch with unequal prompt lengths.
6. Mixed prefill and decode work.
7. Prefix reuse versus fresh execution.
8. Request cancellation and immediate cache reuse.

Exercise lengths:

```text
1, 2, 63, 64, 65,
1023, 1024, 1025, 1026,
8192, 32768, 65536, 131072
```

At the full-context boundary, use a prompt/output split that remains within 131,072 total tokens.

Include selected numerical comparisons at high absolute positions. A successful allocation alone is not evidence that full-context attention is correct.

### 5.5 Numerical release gates

Use a frozen corpus of at least **10,000 teacher-forced token positions**, including:

- Mathematical prompts.
- Reasoning continuations.
- LaTeX.
- Long-context samples.
- Local-window boundaries.
- Both MUDD layers.
- EOS-adjacent positions.

Initial release requirements:

- All compared logits finite for finite qualified inputs.
- Mean absolute softcapped-logit error no greater than **0.02**.
- 99th-percentile absolute softcapped-logit error no greater than **0.15**.
- Mean reference-to-Vulkan distribution KL divergence no greater than **0.001**.
- 99th-percentile per-position KL divergence no greater than **0.01**.
- At least **99.9%** top-token agreement where the reference’s top-two softcapped-logit margin exceeds **0.1**.

These are proposed acceptance thresholds, not measured results.

If the implementation fails them:

- Diagnose the responsible operation.
- Fix arithmetic, masking, precision or cache handling.
- Re-run the affected and downstream tests.
- Do not loosen thresholds merely to obtain a pass.

Keep results separate for Windows and Linux.

### 5.6 Serving/API tests

Require tests for:

- Checkpoint defaults and explicit overrides.
- Greedy decoding.
- Every declared sampler parameter.
- Fixed-seed repeatability.
- Scheduling-independent random streams.
- Streaming/non-streaming text agreement.
- Multiple choices.
- Log-probability semantics.
- Prompt log probabilities.
- EOS versus stop-string termination.
- Stop strings spanning multiple tokens.
- Unicode boundaries.
- Cancellation.
- Invalid requests.
- Context overflow.
- Queue saturation.
- Worker crash and device loss.
- Actual fingerprint checks.
- Disabled reasoning parsing.
- Reasoning delimiters surviving special-token skipping.

Use the Python OpenAI client against the public endpoint as an integration test.

### 5.7 Mathematical quality qualification

Use the repository’s `math-extended` suite and pinned rendering/scoring protocols.

Keep fixed:

- Official checkpoint revision.
- Dataset revisions.
- Prompt-rendering bytes.
- Tokenizer.
- Stop/terminator policy.
- Generation budgets.
- Sampling settings.
- Rollout-group sizes.
- Scorer versions.

The suite includes AIME, MATH-500, GSM8K, HMMT, OlympiadBench, BeyondAIME and Apex Shortlist variants. Preserve the evaluator’s existing contract rather than reconstructing benchmark settings from model-card percentages. [Evaluation contract](https://github.com/paradigma-inc/limite-violetto/blob/7ddad34f5e3d25ddea68c6745fc00373b82bca21/limite-evals/docs/contract.md).

Acceptance requirements:

- Complete the required tasksets.
- Preserve unavailable grader results as unavailable.
- Report truncation and transport failures separately from incorrect answers.
- Compare CUDA and Vulkan by problem, including uncertainty.
- Require the 95% upper confidence bound on overall quality loss to remain within **2 percentage points**.
- Investigate any taskset showing a statistically supported loss greater than **5 percentage points**.
- Increase rollout coverage when uncertainty prevents a conclusion.
- Keep support qualification pending when evidence is insufficient.

Published model-card scores are context, not a directly comparable pass threshold.

Qualify Windows and Linux separately. Shared source code does not prove equivalent driver behavior.

### 5.8 Hardware, stability and performance tests

Mandatory hardware:

- RX 6900 XT on Windows.
- RX 6900 XT on Linux.
- A compatible NVIDIA qualification runner for the pinned CUDA reference.

Software Vulkan may run API, shader and small numerical tests in CI, but must never produce the RX 6900 XT support claim.

Required stability tests:

- At least **5,000 mixed-length requests** with completion and cancellation.
- A **4-hour mixed-workload soak**.
- Repeated load/unload cycles.
- Full-context prefill and decode.
- Four concurrent shorter-context sequences.
- Allocation pressure and segmentation.
- Worker termination during generation.
- Validation-layer runs on small and representative workloads.
- Verification that model-operation CPU fallback remains zero.

Report:

- Time to first token.
- Prompt tokens per second.
- Decode tokens per second.
- Inter-token latency.
- Peak memory.
- Startup and preparation time.
- Throughput at active sequence counts 1, 2 and 4.
- Full-context behavior.
- Dispatch-duration distribution.

Do not promise a specific tokens-per-second target before measurement. BF16 emulation and an H100 reference have very different hardware characteristics.

After qualification, treat regressions exceeding **10%** on the same hardware, driver and workload as requiring investigation.

### 5.9 Implementation milestones

| Milestone | Deliverable | Exit condition |
|---|---|---|
| **M0 — Reproducible foundation** | Pinned sources, separate environments, doctor, reference bundle tooling | Versions and actual device capabilities verified |
| **M1 — Artifact and CPU graph** | Lossless converter, complete native Limite graph, CPU diagnostic implementation | Tensor inventory and architecture fixtures pass |
| **M2 — Vulkan operations** | BF16 primitives, projections, normalization, custom Limite operations | Required operation coverage complete; primitive tests pass |
| **M3 — Cached inference** | Hybrid K/V cache, tiled attention, prefill and decode | Numerical and chunk/cache parity gates pass |
| **M4 — Serving** | API facade, worker protocol, batching, sampling, streaming and cancellation | Public API integration suite passes |
| **M5 — Evaluation and platforms** | Backend-aware evaluator, Windows/Linux binaries | Full-context and mathematical quality qualification pass on both platforms |
| **M6 — Supported release** | Versioned binaries, manifests, documentation and benchmark report | Stability gates pass and reproducible release artifacts are published |

Each milestone must produce inspectable evidence. Completing M2 or generating one plausible answer does not complete the project.

### 5.10 Release contents and support statement

Ship:

- Windows and Linux native Vulkan binaries.
- A separately locked Python package.
- Source and patch-series revisions.
- Compiled shader hashes.
- Converter and artifact-schema versions.
- Reference-table hashes.
- Installation and troubleshooting documentation.
- API compatibility matrix.
- Numerical and benchmark qualification reports.
- Tested driver/platform fingerprints.
- Apache-2.0 and inherited dependency notices.

Retain the original CUDA workflow and verify it has not regressed.

The release may claim:

> Limite 1B Violetto inference and serving are supported on RX 6900 XT through Vulkan on the qualified Windows and Linux configurations, using the documented BF16-semantics numerical profile.

It may make that claim only after the listed hardware and quality gates pass.

Direct `AutoModelForCausalLM.generate()` on a Vulkan PyTorch device, training, LoRA, tensor parallelism and pipeline parallelism are outside this agreed compatibility boundary. The model card itself limits formal support to inference and rejects several sharding and cache/backend combinations. [Supported model interfaces](https://huggingface.co/paradigma-inc/limite-1b-violetto#run-with-transformers).

The deliverable is a fork that preserves the model and its serving/evaluation workflow through a fully implemented Vulkan engine, with compatibility established by evidence rather than by dependency substitutions.
