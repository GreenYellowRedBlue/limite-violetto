---
status: accepted
---

# Preserve the original CUDA path alongside Vulkan

The existing repository serves Violetto through CUDA/vLLM and a locked Linux/NVIDIA environment. We will retain that implementation and environment while adding RX 6900 XT Vulkan support on native Windows and Linux, preserving the model, serving API compatibility, and BF16 semantics. Replacing the original runtime would break the agreed CUDA compatibility path and remove a retained comparison implementation; silently substituting FP16 or quantized model values would change the numerical contract rather than meet it.

This records the requirements explicitly reaffirmed during design review, not measured Vulkan compatibility. The patched-engine boundary, packaging, canonical numerical reference, and tokenizer ownership remain decisions to confirm; see [the design review](../specs/limite-vulkan-design-review.md).
