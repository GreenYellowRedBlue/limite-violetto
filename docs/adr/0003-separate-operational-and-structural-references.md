---
status: accepted
---

# Separate operational and structural numerical references

New Vulkan qualification targets Violetto source snapshot `9f4d1567975a44c35127503deadfe52d434d734f`. The original root CUDA/vLLM 0.26.0 environment is the operational numerical reference; independently pinned native Transformers is a structural/debugging reference, not an interchangeable universal oracle. The user accepted this hierarchy because native eager attention, CUDA SDPA, and vLLM kernels can differ in intermediate rounding and reductions even when they use the same weights.

Keep the evaluator's `b1f3d572ccacb6919f4d64c321b70ba034ddaef2` snapshot, newer engine environment, and historical artifacts explicitly identified rather than silently relabelled. Capture and investigate reference disagreements instead of choosing whichever result passes. No reference run was performed by accepting these decisions.

The user accepted ordinary supported CUDA serving as the operational capture profile, without forcing eager execution. Eager and instrumented captures remain separately labelled diagnostics. Observe and freeze the actual attention backend, compilation/CUDA-graph mode, precision settings, hardware, and driver; the environment lock and source revision do not alone identify that execution.

Versioned reference-derived RoPE and bounded-domain BF16 nonlinear lookup tables are the initial correctness strategy instead of trusting driver transcendental approximations first. Record generator, source snapshot, producer settings, and hashes, and verify rounding and exceptional values before qualification. Tables do not replace GEMM, attention, or end-to-end checks; actual producer settings and reference repeatability remain measurement obligations.

The user accepted exact round-to-nearest-even conversion for non-NaNs, including signed zero, infinities and subnormal boundaries, with arithmetic conversion NaNs canonicalized to quiet BF16 `0x7FC0` and NaN classification compared where reference payloads are unspecified. This avoids claiming unmeasured CUDA payload-bit identity. Preserve serialized checkpoint bytes independently of arithmetic canonicalization; nonfinite model logits from finite qualified inputs still fail qualification.

Retain the brief's section 5.5 initial numerical ceilings and minimum 10,000 teacher-forced positions, including long-context, local-window and MUDD coverage. Freeze the corpus, producers and metric definitions before judging a Vulkan candidate, measure operational CUDA repeatability first, and report Windows/Linux separately. An unusable reference comparison requires investigation, not wider thresholds or selection of a more favorable oracle; no accuracy or repeatability result has been measured.
