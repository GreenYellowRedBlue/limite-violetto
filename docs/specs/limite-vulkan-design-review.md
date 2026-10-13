# Limite Vulkan design review

Evidence and design decisions, starting 2026-10-13. The existing brief is [`limite-vulkan-spec.md`](limite-vulkan-spec.md); the requested `limite-vulkan.md` is absent. The brief remains unchanged. The user accepted Q1-Q21 across four rounds; the design-policy frontier is settled, not the numerical, performance or support claims.

## Fixed requirements

RX 6900 XT; Vulkan; BF16 semantics; native Windows and Linux; model and serving API compatibility; preservation of the original CUDA path. No production implementation begins during this interview. [ADR-0001](../adr/0001-preserve-cuda-alongside-vulkan.md) records the agreed additive boundary; the [glossary](../../GLOSSARY.md) records source-supported terminology rather than implementation choices.

## Source identity is not one pin

The actual checkout is `7ddad34f5e3d25ddea68c6745fc00373b82bca21`, matching the brief. `origin` is `GreenYellowRedBlue/limite-violetto`; `upstream` is `paradigma-inc/limite-violetto`. The local `main` branch still tracks `upstream/main`; tracker operations must continue explicitly targeting the fork.

| Input | Brief or published serving workflow | Actual evaluator |
| --- | --- | --- |
| Violetto source revision | `9f4d1567975a44c35127503deadfe52d434d734f` | `b1f3d572ccacb6919f4d64c321b70ba034ddaef2` |
| CUDA runtime environment | Root lock / vLLM 0.26.0 | Evaluator lock / vLLM 0.28.0, CUDA 12.9 wheel build |
| Native model implementation | HF code in the brief's source revision | Different HF code in the allowlisted revision |

The [actual evaluator allowlist](../../limite-evals/src/limite_evals/limite_models.py#L43-L48) pins the older revision. First-party metadata for [the brief snapshot](https://huggingface.co/api/models/paradigma-inc/limite-1b-violetto/revision/9f4d1567975a44c35127503deadfe52d434d734f?blobs=true) and [the evaluator snapshot](https://huggingface.co/api/models/paradigma-inc/limite-1b-violetto/revision/b1f3d572ccacb6919f4d64c321b70ba034ddaef2?blobs=true) reports identical weight LFS SHA-256, config, tokenizer assets, chat template, and generation config, but different `configuration_limite.py`, `modeling_limite.py`, and model-card blobs. That supports distinguishing weight/text asset identity from native implementation identity, not silently treating the two complete snapshots as interchangeable. The full weights were not downloaded or independently hashed in this review.

The [root manifest](../../pyproject.toml) and [README](../../README.md#L19-L41) describe the original CUDA environment. The [evaluator manifest](../../limite-evals/pyproject.toml#L19-L38) deliberately selects a different engine environment. Its [CLI](../../limite-evals/src/limite_evals/cli.py#L914-L920) permits an explicit engine environment and expected vLLM version, but this does not override the model allowlist or prove numerical equivalence.

## Native engine boundary

The proposed [llama.cpp SHA](https://github.com/ggml-org/llama.cpp/commit/296538adea19ff397100bf8e7d71f302d2d2bb35) resolves. The fork contains the existing Python/vLLM integration and evaluator packages, but no native engine checkout, patch series, Vulkan runtime package, worker, converter, or qualification bundle.

The pinned [conversion registry](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/conversion/__init__.py) does not admit `LimiteForCausalLM`. Checkpoint conversion alone is therefore not an executable model integration. Engine baseline suitability and complete Limite Vulkan coverage remain unproven.

The pinned [server documentation](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/tools/server/README.md#L489-L607) already documents integer-token prompt arrays on `/completion`, raw generated IDs through `return_tokens`, streaming, and configurable sampler chains. We should not assume a brand-new token-input server is necessary. Conversely, those capabilities do not establish Limite's fingerprint handshake, cancellation/termination contract, canonical decoding, or vLLM-compatible sampling semantics. Upstream defaults also enable prompt caching and use different temperature, top-k, and min-p settings; inheriting them would alter the intended request behavior.

**Accepted:** exact upstream pin, auditable ordered patches, isolated Vulkan build/package, a verified immutable upstream source input reusable from an offline build cache, and platform-specific Python wheels containing pinned workers/shaders for native Windows 11 and Ubuntu 24.04 x86-64 ([ADR-0002](../adr/0002-maintain-pinned-limite-patches.md)). **Still open or unmeasured:** wheel ABI/dependency details and observed toolchain/platform fingerprints. The candidate SHA is not qualified.

## BF16 and the numerical reference

The source supports specific operations and casts: [RMSNorm and rotary](../../src/limite_vllm/_components.py#L25-L32), [ordered MUDD mixing](../../src/limite_vllm/_components.py#L80-L106), [scalar folding](../../src/limite_vllm/modeling.py#L157-L161), [XSA correction narrowing](../../src/limite_vllm/modeling.py#L139-L152), and [BF16 raw head projection followed by FP32 softcapping](../../src/limite_vllm/modeling.py#L356-L374). The model body is compile-enabled. Source-level dtype expressions alone do not prove the actual compiled intermediate rounding or reduction behavior.

The pinned [native Transformers implementation](https://huggingface.co/paradigma-inc/limite-1b-violetto/blob/9f4d1567975a44c35127503deadfe52d434d734f/modeling_limite.py) exposes distinct eager, SDPA, and external attention paths. Eager attention narrows softmax probabilities to the query dtype; its CUDA one-token SDPA branch may explicitly select FlashAttention. The fork delegates attention to a vLLM backend. These are complementary implementations, not one interchangeable oracle.

The pinned [GGML precision declarations](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/ggml/include/ggml.h#L1445-L1480) specify minimum accumulator precision and permitted source-conversion ranks. Neither setting establishes Limite's required BF16 rounding after particular operations. BF16 cache support in upstream attention also does not establish model-wide BF16 arithmetic.

**Accepted:** qualification targets `9f4d156…`; ordinary CUDA/vLLM 0.26.0 is operational reference, diagnostics are separately identified, and reference-derived tables are the initial correctness strategy. Exact non-NaN conversion and canonical arithmetic NaNs are explicit; retain the brief's initial numerical ceilings and 10,000-position minimum, freezing the corpus/producers/metrics before judging a candidate ([ADR-0003](../adr/0003-separate-operational-and-structural-references.md)). Producer fingerprints, reference repeatability and actual accuracy remain unmeasured. Do not select whichever reference passes.

## Tokenizer and serving fidelity

The pinned [tokenizer config](https://huggingface.co/paradigma-inc/limite-1b-violetto/blob/9f4d1567975a44c35127503deadfe52d434d734f/tokenizer_config.json) marks `<think>` and `</think>` as non-special added tokens. The [chat template](https://huggingface.co/paradigma-inc/limite-1b-violetto/blob/9f4d1567975a44c35127503deadfe52d434d734f/chat_template.jinja) supplies a fixed mathematical system message and rejects a different system message, tool calls, unsupported roles, and non-string content.

A throwaway diagnostic executed the published `tokenizer.json` using Tokenizers 0.22.2 and the published template using Jinja2 3.1.6 in an isolated `uv run --no-project` environment:

- Observed tokenizer SHA-256: `296e081e2f5ecf9d87814aa9b0f4b12d670ed2b2e2be6c84e01a9466c953afb7`, matching the published LFS identity.
- Vocabulary size: 151,667. Reasoning token IDs: 151665 and 151666.
- Decoding the structural markers around those IDs with special-token skipping retained exactly `<think></think>`.
- A Unicode/LaTeX reasoning string round-tripped unchanged.
- IDs 151667 and 151679 have no token string in that vocabulary.
- The canonical `Solve 2x + 3 = 11.` conversation rendered the fixed system message and assistant generation prefix, producing 48 prompt tokens.
- A custom system message was rejected with the template's declared error.

This is tokenizer/template evidence only, not inference, streaming, sampler, or API qualification. The [model config](https://huggingface.co/paradigma-inc/limite-1b-violetto/blob/9f4d1567975a44c35127503deadfe52d434d734f/config.json) has 151,680 output rows. The 13 undefined tokenizer IDs must not be silently masked, removed, or decoded differently from the selected reference.

**Accepted:** Python/HF owns rendering, encoding, decoding and text stops; native execution supplies raw token-ID events independent of text filtering, with bounded run-ahead/backpressure and an acknowledged terminal prefix/cleanup ([ADR-0004](../adr/0004-python-owns-canonical-text.md)). Preserve the complete declared serving surface and same-numerical-profile text repeatability, without promising cross-profile text identity ([ADR-0006](../adr/0006-preserve-declared-serving-semantics.md)). Interface bounds and runtime conformance still require specification/evidence.

## Hybrid attention and cache

The [pinned config](https://huggingface.co/paradigma-inc/limite-1b-violetto/blob/9f4d1567975a44c35127503deadfe52d434d734f/config.json) and [actual window translation](../../src/limite_vllm/_components.py#L153-L172) support 12 global layers, 36 local layers, and a local distance of 1,024, meaning 1,025 visible keys including the query. Global layers bypass rotary. The current [attention path](../../src/limite_vllm/modeling.py#L126-L144) passes value-embedding-adjusted V to attention and uses that same current V for XSA. [MUDD history](../../src/limite_vllm/modeling.py#L281-L305) is residual state at earlier layer depths for each token, not temporal K/V history.

The pinned [llama.cpp mixed-window cache](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/src/llama-kv-cache-iswa.cpp) already separates non-sliding and sliding layers, operates sequence copying/removal on both components, and sizes the sliding component using window length plus microbatch capacity, padded to 256. This is not evidence for the brief's proposed bespoke 64-token page allocator or its aligned-memory estimate.

**Accepted:** adapt the existing split cache and allocator before introducing a separate pager; prefix reuse remains disabled until full-state parity passes ([ADR-0007](../adr/0007-adapt-native-hybrid-cache.md)). Preserve mask/content/absolute-position invariants. Page/slab constants, chunks, admission counts and memory estimates remain candidates. The configured 131,072-token maximum is not measured full-context support.

## Evaluator integration and quality claims

The [measurement contract](../../limite-evals/docs/contract.md) fixes model artifacts, exact rendering, vocabulary, terminators, stops, datasets, scorer, sampling, rollout groups, and generation budgets. [Runner requests](../../limite-evals/src/limite_evals/runner.py#L243-L329) use non-streaming chat, explicit special-token skipping, optional stop-token IDs and sampler fields, and require raw completion text plus termination/usage evidence. [Reasoning-parser rejection](../../limite-evals/src/limite_evals/runner.py#L167-L193) is an existing consumer invariant. Streaming and log-probability support need separate serving qualification; the evaluator does not exercise them today.

The [current launcher](../../limite-evals/src/limite_evals/serve.py#L349-L443) selects POSIX vLLM executables and [gates the observed engine](../../limite-evals/src/limite_evals/serve.py#L297-L315). A backend adapter is substantive work, not a base-URL or version-string substitution. CUDA-only dependencies are platform-marked but still included by default on Linux. A portable scoring/launch arrangement must not quietly install CUDA for Linux Vulkan evaluation.

**Accepted:** one measurement contract and engine adapters, isolated dependencies, preserved legacy/CUDA behavior, plus strict cross-engine qualification over verified source/lossless derivation and frozen actual inputs ([ADR-0005](../adr/0005-preserve-one-evaluation-contract.md)). Use equal-taskset expected headline loss on the complete frozen corpus, randomized paired seeds, the independent-cluster Hoeffding bound and one predeclared claim-bearing analysis per platform/candidate ([ADR-0008](../adr/0008-use-equal-taskset-quality-loss.md)). Concrete manifest/launcher integration, justified dependence provenance and actual coverage remain unimplemented or unmeasured.

The accepted overall gate is a one-sided 95% upper bound no greater than two percentage points; an unresolved statistically supported taskset loss above five points blocks release even if the aggregate passes. Predeclare multiplicity-aware taskset checks and complete coverage. [Existing suites](../../limite-evals/src/limite_evals/suites.py) have different rollout/group sizes, scorer availability and known overlaps. Additional claim-bearing looks require predeclared error allocation or a valid sequential method; do not keep adding samples until a favorable ordinary interval appears.

## Discoverable hardware and resource access

Runtime probes executed in this review:

- Windows `Win32_VideoController` identified an AMD Radeon RX 6900 XT, device `1002:73bf`, Windows driver version `32.0.21030.2001`.
- `vulkaninfo --summary` succeeded and reported device API 1.4.315 and driver information `25.10.30.02`.
- The full `vulkaninfo` run emitted 2 GiB maximum individual allocation and buffer sizes, 32 KiB compute shared memory, and subgroup range 32-64, but timed out after 45 seconds. Do not treat the partial dump as complete cooperative-matrix/BF16 capability evidence. The loader also reported ReShade/RTSS layer-version warnings and a ReShade DLL-load error; a clean qualification environment remains unestablished.
- `nvidia-smi` failed with an insufficient-permissions error. The Windows adapter enumeration showed only AMD; no usable local CUDA reference runner was established.
- WSL lists a stopped Ubuntu distribution. This is not qualification evidence for native Linux Vulkan on the RX 6900 XT.
- The fork's GitHub Actions runners endpoint returned zero self-hosted runners. No SSH hosts or OpenSSH config were discoverable through the checked host inventory/config path.
- `glslc`, `spirv-val`, `cmake`, MSVC `cl`, and `gcc` were not on this process's PATH. This does not prove they cannot be installed or exist elsewhere.

**Accepted hardware route:** operator-provided Linux/NVIDIA reference access and native Linux RX 6900 XT access, alongside Windows. No permission to rent hardware or change this workstation's boot configuration is implied. Access and operator handoff remain operational prerequisites, not observed capabilities. No GPU inference/reference capture, native-engine build, full mathematical suite, or platform qualification was performed.

## Decision status and qualification prerequisites

The user accepted Q1-Q21 across four rounds. ADRs 0002-0008 record hard-to-reverse choices; no user-policy question remains open in this interview. Hardware access and measured qualification remain prerequisites, not observed capabilities.

Still required before qualification:

- Numerical reference -> observed producer fingerprints, reference repeatability and frozen corpus/metric details under accepted gates.
- Serving interface -> source-derived conformance matrix, versioned token-control interface, calibrated run-ahead bounds and same-profile repeatability evidence.
- Quality endpoint -> instantiate the accepted seed/cluster experiment with actual dependence provenance, complete frozen coverage and claim-family error allocation under the one-look policy; do not relabel legacy deterministic runs.
- Distribution -> wheel ABI/dependencies, artifact provenance, observed platform/toolchain/driver fingerprints and operator handoff.
- Hardware/validated engine -> full-context, batching, dispatch-duration and release evidence. Source-defined facts are not substitutes for these runs.

This closes the design-policy interview without authorizing production implementation. Concrete interfaces, numerical producers, packaging and evidence collection remain engineering obligations under the accepted policies, and the original brief is not measured proof.

## Round-two evidence

The following observations inspect existing code or exercise existing CPU-side contracts. They are not a native-engine build, model inference result, or GPU qualification.

### Reference execution mode is a measurement input

The model body is decorated for compilation in [`modeling.py`](../../src/limite_vllm/modeling.py#L218-L225). The evaluator exposes [`--enforce-eager`](../../limite-evals/src/limite_evals/cli.py#L947-L951) as an opt-in change to CUDA graph/compile execution, and [`RunManifest.comparable_with`](../../limite-evals/packages/limite-evals-core/src/limite_evals_core/schema.py#L627-L630) distinguishes that mode. The original locked environment is PyTorch `2.11.0+cu130`, Transformers `5.6.2`, and Tokenizers `0.22.2`; the source snapshot and environment alone do not identify the actual attention backend or compiler/kernel settings of a run.

**Accepted Q7-Q8:** ordinary supported CUDA serving for operational logit/quality captures; eager/instrumented execution is separately labelled diagnostic. Observe and freeze execution fingerprints. Versioned reference-derived tables are the initial correctness strategy; producer settings still need actual capture records. Q14-Q15 below settle exceptional-value policy and initial numerical gates without supplying reference-repeatability evidence.

### Existing native streaming can omit token-ID events

In the pinned [`server-context.cpp`](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/tools/server/server-context.cpp), `process_token` records the sampled token, then checks decoded UTF-8 completeness. It calls `add_token` and sends a partial response only when text is complete. `send_partial_response` assigns the current token ID, while stream-mode `send_final_response` supplies an empty token list. Consequently this path is not an independent, complete token-ID event channel: IDs for incomplete text pieces can be withheld without later being supplied as the pending token sequence.

Sampling acceptance and the native generation counter also precede text-stop/EOS handling. Returned IDs, emitted text, termination, and usage cannot be inferred from one another. Explicit task cancellation releases a slot, but that does not itself define a Python/native terminal-prefix or stop-acknowledgement contract.

A new tokenizer diagnostic used the actual pinned Tokenizers 0.22.2 implementation. The Gothic character `𐍈` encodes as `[126190, 230]`; decoding those IDs together returns `𐍈`, while decoding each separately returns two replacement characters. This demonstrates a concrete need for complete token-ID provenance and incremental decoding. A compatibility ideograph also normalized under the tokenizer; arbitrary input-byte identity is not the tokenizer contract. These observations do not claim an executed failure in a Limite Vulkan worker, which does not yet exist.

**Accepted Q10:** an independent raw token-ID event channel, bounded run-ahead/backpressure and an acknowledged terminal prefix/cleanup instead of a separate HTTP acknowledgement after every token. Only Python matches text stops. API usage and reusable state describe the accepted prefix; discarded run-ahead work remains separately observable. The actual bound and control-message interface are not measured or implemented.

### Cache reuse has both existing machinery and precise limits

The pinned [`llama-kv-cache-iswa.cpp`](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/src/llama-kv-cache-iswa.cpp) prepares both global/base and sliding caches for each microbatch. Its partial-only state serialization intentionally skips the global/base cache; full prefix restoration for Limite therefore cannot be obtained from a partial-only snapshot alone.

Further allocation tracing narrows the earlier open question. [`ggml-alloc.c`](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/ggml/src/ggml-alloc.c#L1147-L1158) hands static tensor lists to the backend allocator. [`ggml-backend.cpp`](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/ggml/src/ggml-backend.cpp#L56-L181) already partitions those lists at tensor boundaries using the buffer type's advertised maximum size. It does not split a single oversized tensor. The dynamic graph allocator also has a finite chunk count and paths that permit larger chunks; generic splitting is not a blanket allocation-limit proof.

The pinned [`Vulkan buffer implementation`](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/ggml/src/ggml-vulkan/ggml-vulkan-buffers.cpp) rejects a buffer larger than its device limit. The [`Vulkan device setup`](https://github.com/ggml-org/llama.cpp/blob/296538adea19ff397100bf8e7d71f302d2d2bb35/ggml/src/ggml-vulkan/ggml-vulkan.cpp#L4186-L4203) records allocation/buffer limits and offers a forced maximum-buffer-size diagnostic setting. Actual Limite tensors, alignment, views, memory admission, and graph workspace remain unqualified on the target.

**Accepted Q9:** adapt the existing split cache and allocator first, preserving model/state invariants. The 64-token pages and 512 MiB slabs remain candidates. Prefix reuse is disabled until full-state parity passes; full-context and public serving targets are unchanged.

### The existing comparison verdict is not strict qualification

A throwaway `uv run --no-project` diagnostic loaded the actual schema with Pydantic `2.13.4` (the evaluator lock's version) and constructed synthetic manifests, not evaluation runs:

- Different artifact and template fingerprints returned no generic comparison rejection reasons.
- Different stop strings also returned no rejection reasons.
- Different eager modes returned `different vLLM eager mode: False vs True`.

The implementation intentionally supports broader model/profile comparisons; the textual evaluation contract is stricter about same-model input identity. Reusing the generic verdict unchanged would therefore not enforce the Vulkan qualification assumptions. The resolver's [Hub identity hash](../../limite-evals/src/limite_evals/resolve.py#L273-L274) is a hash of an identity string, not a full-weight byte verification, and its [local artifact hash](../../limite-evals/src/limite_evals/resolve.py#L259-L270) covers config and top-level Safetensors, not an independently verified converted artifact.

**Accepted Q11:** a dedicated strict cross-engine qualification policy requiring the same verified source snapshot and lossless derivation, actual text assets, frozen benchmark/rendered inputs, scorers, budgets, sampling, stops and decode policy. Keep runtime-artifact byte hashes distinct and allow only declared engine/build/layout differences; do not globally relax the generic comparator.

The suite also has an existing data-pinning gap: [`gsm8k` has no source revision](../../limite-evals/src/limite_evals/suites.py#L343-L355), [`_rows` omits the revision argument](../../limite-evals/src/limite_evals/tasksets.py#L276-L291) for that case, and the [manifest records `unpinned`](../../limite-evals/src/limite_evals/cli.py#L722-L739). New qualification must freeze the actually resolved data and rendered inputs rather than claim that this declaration alone pins every dataset. Historical manifests must not be rewritten to pretend they contained that evidence.

### The quality endpoint is fixed; uncertainty remains open

The [headline selector](../../limite-evals/src/limite_evals/aggregate.py#L169-L193) uses each taskset's declared anchor with `reference` fallback when no published `exact` result is available; relaxed scorer views are not promoted to the headline. [Existing statistics](../../limite-evals/packages/limite-evals-core/src/limite_evals_core/stats.py#L52-L59) average rollout outcomes within each problem, then average problems. [Paired comparison](../../limite-evals/src/limite_evals/compare.py#L105-L131) shares problem draws between engines but resamples rollouts separately, uses 2,000 resamples and a two-sided 95% interval, and defines the delta as its first run minus its second.

A diagnostic of the existing statistics on explicitly synthetic outcomes observed:

| Statistic on one correct single-rollout problem and one incorrect 32-rollout problem | Result |
| --- | --- |
| Equal-problem mean used by the existing helper | 0.5 |
| Pooled-rollout mean | 0.030303030303030304 |

The existing bootstrap on two synthetic homogeneous 32-rollout problems returned point `0.5` and interval `[0.0, 1.0]`. These are illustrative calculations, not model scores; more rollouts do not eliminate uncertainty from a small problem corpus.

The [Apex Shortlist caveat](../../limite-evals/src/limite_evals/suites.py#L477-L500) records overlap with AIME/HMMT. A problem-population uncertainty calculation must not silently treat repeated problems as independent sampled problems. Under the subsequently accepted fixed-corpus scope, shared problem identity alone does not imply dependence between independently redrawn completions; shared random streams or reused outcomes do. The existing per-taskset paired routine does not itself produce an overall suite loss interval or a sequential stopping rule.

**Accepted Q12:** each declared taskset has equal weight in the overall loss, using its existing headline and per-problem rollout policy. Preserve the full problem sets, scorer/grouping and overlap caveats. Confidence construction, dependence, coverage and stopping rules follow this endpoint; an inconclusive result is not qualification.

### Source distribution uses explicit preparation

**Accepted Q13:** checksum-verified immutable upstream source input plus local ordered patches, fetched only during explicit preparation/builds and reusable from an offline build cache. Ordinary serving uses a pinned native binary rather than fetching or compiling an unpinned engine. This records the source-distribution policy, not permission to add a dependency checkout or build pipeline during the interview.

## Round-three evidence and accepted policies

The user accepted Q14-Q19 after the following evidence was inspected. These policies constrain future implementation and qualification; acceptance does not turn source observations into measured GPU results.

### The serialized inventory includes scalar values omitted from the brief's count

A bounded HTTP Range diagnostic read the eight-byte length and 77,048-byte JSON header from the [pinned official Safetensors artifact](https://huggingface.co/paradigma-inc/limite-1b-violetto/resolve/9f4d1567975a44c35127503deadfe52d434d734f/model.safetensors). It required HTTP 206 and matching Content-Range before reading. The total fetched was 77,056 bytes, not the weight payload.

| Header inventory | Tensor count | Serialized values | Payload bytes |
| --- | ---: | ---: | ---: |
| BF16 | 338 | 1,035,141,120 | 2,070,282,240 |
| F32 | 405 | 113,056 | 452,224 |
| Total | 743 | 1,035,254,176 | 2,070,734,464 |

The F32 inventory contains 288 zero-dimensional tensors (`shape: []`), including residual/post multipliers and projection scales. Excluding those scalar values gives the brief's 112,768 F32 count. That explains the arithmetic discrepancy; it does not prove why the published metadata omitted them. Header length plus its eight-byte prefix and declared payload totals the reported 2,070,811,520-byte file length. Observed header SHA-256 was `5cbfe7584878b6ec317f11e9ba9f82e1237808306930d19437942621a8f402c5`.

Conversion inventory must use the actual serialized names, shapes, dtypes and values, including zero-dimensional scalars. The full weight payload was neither downloaded nor byte-hashed here; header arithmetic and published LFS metadata are not a full-weight integrity check or model inference.

### Conversion, scalar arithmetic, and kernels have different evidence

[PyTorch 2.11.0's portable BF16 helper](https://github.com/pytorch/pytorch/blob/v2.11.0/torch/headeronly/util/BFloat16.h) implements non-NaN round-to-nearest-even using integer bits and canonicalizes FP32 NaNs to positive quiet BF16 `0x7FC0`. On CUDA architectures SM80 and above, its constructor instead uses `__float2bfloat16`. The [CUDA 13.0 conversion documentation](https://docs.nvidia.com/cuda/archive/13.0.0/cuda-math-api/cuda_math_api/group__CUDA__MATH____BFLOAT16__MISC.html) specifies round-to-nearest-even but does not establish the intrinsic's actual NaN payload/sign or GEMM denormal behavior. No CPU/CUDA NaN-bit disagreement was measured.

The previously collected partial device dump reports FP32 signed-zero/Inf/NaN preservation, denormal preservation, denormal flush-to-zero, and round-to-nearest-even support, with rounding and denormal independence `32_BIT_ONLY`. [Khronos defines these as supported execution modes](https://docs.vulkan.org/refpages/latest/refpages/source/VkPhysicalDeviceFloatControlsProperties.html), not proof that a shader enables them or that BF16 conversion/arithmetic is qualified. An integer-bit BF16 primitive, FP32 arithmetic settings, LUT special values, and GEMM/reduction behavior need distinct checks.

**Accepted Q14:** exact round-to-nearest-even non-NaN conversion, including signed zero, infinities and subnormal boundaries; arithmetic conversion NaNs canonicalize to quiet BF16 `0x7FC0`, with NaN classification compared where reference payloads are unspecified. Serialized checkpoint bytes remain lossless independently. Exceptional-value handling must not hide nonfinite model logits from finite qualified inputs.

### Pinned serving source settles details but does not qualify the worker

| Contract detail | vLLM 0.26.0 source observation |
| --- | --- |
| Reported generated log probabilities | The default raw-logprob path computes FP32 log-softmax before penalties, temperature and sampling processors; sampled-token and top-k records use that distribution. |
| Prompt log probabilities | The first prompt token's log probability is unavailable (`None`), not a fabricated zero. |
| Multiple choices with an explicit seed | Each child uses `seed + choice_index` and `n = 1`. |
| Hidden terminal token | EOS/stop-token handling may omit its text while retaining its ID in the completion-token sequence. |
| Completion usage | Non-streaming completion usage sums generated token-ID lengths over choices; prompt tokens are counted per input prompt, not per choice. Echoing prompt text does not make it generated usage. |
| Pretokenized prompt bounds | Input validation uses the larger of tokenizer maximum ID and model vocabulary maximum ID; it does not restrict prompts to defined tokenizer strings. |

Sources: [sampler](https://github.com/vllm-project/vllm/blob/v0.26.0/vllm/v1/sample/sampler.py), [model logprob configuration](https://github.com/vllm-project/vllm/blob/v0.26.0/vllm/config/model.py), [prompt-logprob initializer](https://github.com/vllm-project/vllm/blob/v0.26.0/vllm/logprobs.py), [parallel choices](https://github.com/vllm-project/vllm/blob/v0.26.0/vllm/v1/engine/parallel_sampling.py), [detokenizer](https://github.com/vllm-project/vllm/blob/v0.26.0/vllm/v1/engine/detokenizer.py), [completion serving](https://github.com/vllm-project/vllm/blob/v0.26.0/vllm/entrypoints/openai/completion/serving.py), and [input validation](https://github.com/vllm-project/vllm/blob/v0.26.0/vllm/v1/engine/input_processor.py).

With Violetto's declared 151,680 model rows and tokenizer maximum ID 151666, that input-validation condition permits IDs through 151679. This is a source-derived validation fact, not an executed padded-ID model/decode result. No inspected sampler path justifies adding an unconditional mask for those 13 rows; full operational behavior remains a reference-capture obligation.

API formatting also has separate semantics: the completion formatter maps unavailable first-token log probabilities to null and clamps reported numerical log probabilities at `-9999.0` for JSON-compatible output. Do not confuse this wire representation with the numerical distribution used for comparison or sampling. Freeze the supported parameter/error matrix from the reference instead of asking the user to invent these inherited facts.

**Accepted Q16:** scope identical-text repeatability to the same numerical execution profile, including batch shapes, cache mode and kernels, explicitly narrowing the brief's platform/build-only wording. Random streams remain isolated and scheduling-independent; explicit-seed choices use `seed + choice_index`. A profile change may change logits and text; same-profile repeatability and streaming/non-streaming conformance still require qualification.

### A confidence claim requires a target and stopping policy

The existing [paired comparator](../../limite-evals/src/limite_evals/compare.py#L105-L131) takes two-sided 95% percentile endpoints, so its upper endpoint uses the 97.5th percentile, not a one-sided 95% endpoint. Its problem/rollout bootstrap also embodies problem-population assumptions; it is not automatically the right uncertainty construction for expected stochastic correctness on a fixed benchmark corpus.

Known repeated problems retain their declared taskset memberships and weights. Shared/reused completions, random streams, rollout variants or choices must not be counted as independent without evidence. Under a fixed-corpus claim, repeated problem identity alone is not random dependence if generation draws are genuinely independent. Degenerate bootstrap draws do not prove universal finite-sample coverage, and 2,000 resamples are computation, not 2,000 additional model observations.

[Howard et al., *Time-uniform, nonparametric, nonasymptotic confidence sequences*](https://arxiv.org/html/1810.08240v9) explains why ordinary fixed-sample intervals cannot be monitored repeatedly with arbitrary stopping while retaining their nominal guarantee. The paper provides sequential methods under stated assumptions; it does not validate this repository's bootstrap.

**Accepted Q17-Q18:** the bound concerns expected stochastic correctness on the complete frozen corpus, not unseen problems. Use one predeclared claim-bearing analysis per platform/release candidate after frozen coverage is complete, separate from pilots; apply the one-sided 95% upper-bound criterion with a two-point margin. Additional looks require predeclared error allocation or a valid sequential method. Platform claims are separate, not a joint 95% guarantee. The confidence method follows this scope. Unavailable graders are not incorrect answers, tasksets are not silently dropped, and inconclusive results remain pending.

### Numerical and packaging candidates are not measured support

**Accepted Q15:** retain the brief's 10,000-position corpus minimum and initial numerical ceilings; freeze corpus, producers and metric definitions before candidate qualification, measuring CUDA-reference repeatability first. Do not widen thresholds to pass a candidate. These are accepted initial gates, not observed CUDA noise or Vulkan accuracy.

**Accepted Q19:** the first native targets are Windows 11 x86-64 and Ubuntu 24.04 x86-64, with platform-specific Python wheels containing pinned workers and compiled shaders in a separately locked Vulkan package. End users need a compatible driver/loader, not SDK installation or runtime compilation. The brief's MSVC v143/GCC 13, CMake 3.31.x, SDK 1.4.328.1 and Vulkan 1.3 SPIR-V selections remain unexercised build candidates. Exact qualified fingerprints, wheel ABI/dependency audits and broader-support evidence remain release obligations.

## Round-four prerequisites

### Claim-bearing groups and seed allocation

The [suite composition](../../limite-evals/src/limite_evals/suites.py#L539-L549) contains ten ordinary protocol-headline tasksets, not a native/source-group metric caller. Seven use 32 rollouts, MATH-500 and OlympiadBench use four, and GSM8K uses one. The generic [source-group aggregate](../../limite-evals/src/limite_evals/aggregate.py#L551-L586) treats variants as fixed rows rather than stochastic repetitions; its presence is not evidence that `math-extended` uses it.

The [runner formula](../../limite-evals/src/limite_evals/runner.py#L400-L410) is `base_seed + problem_index * group_size + rollout_index`, and the [task invocation](../../limite-evals/src/limite_evals/cli.py#L513-L524) passes the same base seed to each taskset. Thus seed values are reused across tasksets at matching ordinal offsets; the taskset name is not a namespace. Distinct offsets are deterministic allocation, not evidence of independently randomized draws. The [persisted sample fields](../../limite-evals/src/limite_evals/runner.py#L418-L438) do not include the derived seed.

The existing caveat explicitly identifies four Apex problems also counted in AIME 2025/HMMT 2025. That fixes a weighting caveat, not a universal stochastic correlation map. No general cross-taskset source identity or independent-stream provenance was established in the inspected claim-bearing paths. Independent seed assignment, completion reuse, numerical execution coupling and larger dependence clusters therefore need an explicit qualification policy; historical results must not be relabelled as randomized.

### The accepted text boundary overrides the brief's candidate worker fields

The unchanged brief's section 2.3 lists native stop strings, detokenization policy and stable text-fragment events. ADR-0004's accepted ownership instead makes Python authoritative for text stops/decoding and requires independent raw token-ID events. Future protocol design must follow that ADR rather than implement those candidate fields as a second text engine. Keep the brief's loopback/per-launch authentication and version/launch-identity checks; actual message schemas and bounded-credit sizes remain implementation/measurement obligations, not permission to duplicate text ownership.

### A synthetic dependence diagnostic is not a model result

A throwaway standard-library calculation exercised a proposed weighted bounded-difference upper-bound formula. It enumerated the exact binomial distribution for 20 independent symmetric paired differences in `{-1,+1}` with true mean loss zero, and contrasted 100 perfectly dependent copies of one such difference:

| Synthetic calculation | Observed result |
| --- | ---: |
| Independent 20-pair one-sided radius at alpha 0.05 | 0.5473328305111974 |
| Exact probability that this independent upper bound misses zero | 0.005908966064453125 |
| Incorrect independence-assuming upper bound for 100 shared `-1` outcomes | -0.7552253169319183 |
| Miss probability under that shared-outcome construction | 0.5 |
| Correct single-cluster bound, capped at maximum possible loss | 1 |

The example demonstrates why an independence-assuming radius cannot be transferred to shared outcomes. It does not establish the actual seed/engine independence, validate repository bootstrap coverage, measure model quality or select the confidence procedure. Conservative distribution-free bounds can require more coverage; no sample-count or runtime promise follows from these synthetic cases.

### Accepted final policies

The [confidence research memo](limite-vulkan-confidence-research.md) derives the accepted conservative weighted independent-cluster Hoeffding bound from primary sources, contrasting it with approximate fixed-corpus bootstrap and independent whole-suite replication. The experiment retains all fixed problem memberships and weights, assigns independently randomized paired request seeds in an immutable prospective manifest, and uses only clusters whose independence is defensible. Coverage is over that declared randomization/execution experiment, not a conditional guarantee for every fixed seed list. Actual profile/reset rules and seed-induced sampling law must match the claim; the current deterministic evaluator cannot simply be relabelled.

**Accepted Q20:** use that qualification-only experiment and finite-sample bound, preserving historical/default CUDA seed behavior. Freeze candidate, seed distribution, corpus, cluster/weight map, complete coverage and analysis before claim-bearing outputs. Use the smallest justified independent units, merge actual shared-outcome/randomness/state dependencies, and retain pending status when an informative bound is unsupported. Ordinary operational CUDA mode remains the reference; moving to an isolated execution profile must be explicit, not a favorable diagnostic substitution. The cost may exceed approximate-bootstrap qualification; no GPU/sample-count promise is made.

**Accepted Q21:** a statistically supported taskset loss above five percentage points blocks release while unresolved, even if the overall macro bound passes. Predeclare multiplicity-aware taskset checks; the conservative default is Bonferroni-adjusted one-sided lower bounds across the fixed taskset family. Fix and requalify rather than waive the regression through aggregate compensation. A non-triggering check is not proof that every taskset's loss is below the margin. This is stricter than the brief's investigate-only wording.

The remaining concrete protocol, producer, build and coverage records are engineering/measurement obligations under accepted policies, not additional product choices to invent. Operator-provided Linux/NVIDIA and native Linux RX 6900 XT access is still unprovided; it is a prerequisite for future evidence collection, not a measured capability or permission to begin production implementation.

## Interview outcome and final smoke proof

Q1-Q21 are accepted; no policy question remains open in this interview. The glossary and ADRs record the shared design, while this review and the confidence note distinguish source observations, assumptions and measurements. The interview changed documentation only, preserved the original brief and repository instructions, and did not begin production implementation.

A throwaway standard-library calculation exercised the accepted upper bound and the Bonferroni taskset lower-bound rule under explicitly synthetic independent-unit assumptions:

| Synthetic scenario | Overall upper bound | Severe-taskset lower bound | Quality-rule outcome |
| --- | ---: | ---: | --- |
| Zero observed loss over 20,000 equal-weight independent pairs | 0.017308183826022852 | Not applicable | Overall gate passes |
| Same zero observed loss, but all contributions share one dependence cluster | 1 | Not applicable | Overall gate does not pass |
| Ten tasksets with 100,000 independent pairs each; one observed loss 0.08 and nine zero | 0.010447746830680817 | 0.06970600430683203 | Macro gate passes; severe taskset blocks release |

All three scenario checks passed. They exercise grouping sensitivity and the rule preventing aggregate compensation; they are not actual model scores, seed/engine independence evidence, empirical coverage validation, sample-count recommendations or platform qualification.

Before future qualification, obtain usable operator-provided CUDA reference and native Linux RX 6900 XT access; freeze the actual corpus, seed law/manifest, producer and execution fingerprints, dependence/reset rules, coverage and error accounting; and collect the native numerical, serving, cache, full-context, stability and paired-quality evidence on both target platforms. Accepting this design supplies none of that measured evidence and authorizes no production implementation.
