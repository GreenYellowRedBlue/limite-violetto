# Limite Vulkan

Limite Vulkan brings Violetto's mathematical inference and serving behavior to the RX 6900 XT while retaining the original CUDA path. This vocabulary distinguishes the model's identity and semantics from evidence that an execution path preserves them.

## Language

### Model identity and behavior

**Violetto**:
The post-trained Limite 1B model intended to solve mathematical problems with step-by-step reasoning and a boxed final answer.
_Avoid_: A generic Llama or Qwen model

**Violetto source snapshot**:
An immutable published bundle containing Violetto's weights, architecture configuration, tokenizer, prompting and generation assets, and supplied native model code. Identical weights alone do not establish identical source snapshots.
_Avoid_: Checkpoint hash when referring to the whole bundle

**Canonical mathematical prompt**:
Violetto's fixed mathematical system message and the exact conversation rendering supplied with its source snapshot.
_Avoid_: An approximately equivalent ChatML prompt

**Reasoning delimiters**:
The ordinary text tokens marking the beginning and end of Violetto's reasoning. They remain part of the raw completion even when structural special tokens are omitted.
_Avoid_: Special tokens, separately parsed reasoning

**Canonical completion text**:
Violetto's generated text under the source snapshot's vocabulary and declared special-token and stop policies, with reasoning delimiters retained. It is distinct from grader-extracted answers or separately parsed reasoning.
_Avoid_: Grader input reconstructed from parsed reasoning

**Accepted completion prefix**:
The initial generated-token sequence accepted through the completion's terminal decision. Returned text, termination, usage, and reusable sequence state must describe that same prefix.
_Avoid_: Every token the worker happened to compute

**Completion run-ahead**:
Native generation beyond the prefix Python has accepted while interpreting the completion. Work beyond a terminal prefix is not silently part of the returned completion.
_Avoid_: Accepted completion tokens

**Lossless runtime artifact**:
A derived representation of a Violetto source snapshot that preserves its serialized parameter values and model-defining information.
_Avoid_: Quantized checkpoint, approximately equivalent model

**BF16 semantics**:
Violetto's numerical behavior in which BF16-valued quantities retain the required rounding boundaries while designated FP32 quantities retain their precision. This is neither BF16 storage alone nor a promise of bitwise equality across execution paths.
_Avoid_: BF16 cache support, FP16 approximation, CUDA-bitwise identity

### Attention and residual state

**Local attention distance**:
The maximum separation between a query position and an earlier visible key position.
_Avoid_: Local attention span

**Local attention span**:
The number of visible key positions in a full local window, including the query position. It is one greater than the local attention distance.
_Avoid_: Local attention distance

**Hybrid attention state**:
The sequence state combining full causal history for global layers with the retained local history for local layers.
_Avoid_: A single sliding window for every layer

**Value-embedding-adjusted value**:
A token's value after its gated value-embedding contribution has been added. This is the value used by attention, retained for future attention, and supplied to the current-token XSA correction.
_Avoid_: Raw projected value, attention-weighted value

**MUDD tap**:
A selected residual-stream state at an earlier layer depth for the same token. MUDD taps are not earlier tokens in the sequence.
_Avoid_: Temporal cache entry, previous-token state

### Qualification evidence

**CUDA reference**:
The primary measured execution of the retained CUDA path used to compare operational numerical behavior, identified by its model assets, software environment, hardware, and execution settings. It is distinct from a structural diagnostic or a published benchmark score.
_Avoid_: Universal oracle, model-card baseline

**Violetto structural reference**:
An independently identified native model implementation used to diagnose Violetto's architecture, casts, masks, and operation order. Its results are not automatically interchangeable with the operational CUDA reference.
_Avoid_: A second oracle that can be selected whenever it passes

**Reference-derived numerical asset**:
Recorded numerical values produced under an identified reference execution to preserve Violetto's numerical behavior. Such an asset is not evidence that the complete execution path is qualified.
_Avoid_: Driver capability, unversioned numerical approximation

**Violetto numerical execution profile**:
The identified numerical execution conditions under which Violetto's same-prompt, same-parameter, same-seed completion is expected to repeat. Identical model assets and platform alone do not establish the same profile.
_Avoid_: Platform/build identity alone, seed identity alone

**Evaluation measurement contract**:
The model, rendering, vocabulary, termination, data, scoring, sampling, rollout-group, and generation-budget definitions that determine what a Limite evaluation result measures. Changing engines does not by itself authorize changing this contract.
_Avoid_: Matching benchmark names, matching model names

**Qualification corpus**:
The complete frozen mathematical problem set, with its rendered inputs, taskset memberships and declared scoring and rollout rules, on which Limite's stochastic correctness claim is evaluated. It is not a claim about an unseen-problem population.
_Avoid_: A representative sample of all mathematics, whichever tasksets completed

**Paired qualification rollout**:
Corresponding CUDA and Vulkan completions measured for the same frozen problem, rendering, generation policy and assigned seed. Pairing neither requires identical completion text nor establishes independence.
_Avoid_: Identical-output pair, independent engine samples

**Qualification dependence cluster**:
A set of paired qualification rollouts treated together because their outcomes share randomness, reused measurements or outcome-affecting execution conditions. Independence is a property required between clusters under the declared qualification experiment.
_Avoid_: Every distinct seed, every distinct problem

**Suite quality loss**:
The equal-taskset expected headline correctness loss of CUDA relative to Vulkan on the qualification corpus, retaining each taskset's per-problem rollout policy. A negative loss is a Vulkan gain; an observed point estimate alone is not a qualification verdict.
_Avoid_: Pooled-rollout accuracy difference, most favorable scorer difference

**Validated implementation route**:
An evidence-backed set of decisions and feasibility checks sufficient to plan complete Violetto inference and serving on the required Vulkan platforms. It is not an implemented engine or a platform-support qualification.
_Avoid_: Accepted design alone, successful build alone, qualified runtime

**Platform qualification**:
Evidence that the required model, numerical, serving, and evaluation behavior holds on a specified RX 6900 XT operating-system, driver, and build combination.
_Avoid_: Device detection, successful compilation, plausible output
