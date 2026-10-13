---
status: accepted
---

# Preserve the declared serving semantics

The user accepted the complete capability surface in section 4.1 of the existing Vulkan brief as the serving compatibility target, including streaming, multiple choices, generated and prompt log probabilities, sampler parameters and penalties, stops, cancellation, and token-usage accounting. Matching only the current evaluator's non-streaming chat requests is insufficient: it exercises a subset of the public serving contract.

Compatibility means preserving supported request/response semantics, not every vLLM administrative feature or internal Python interface, and not identical CUDA/Vulkan stochastic text for matching seeds. Freeze explicit parameter, log-probability, termination, and error semantics and qualify them against the retained serving reference rather than accepting and ignoring behavior-changing fields. No conformance run was performed by accepting this ADR.

The user explicitly narrowed the brief's platform/build-only identical-text wording to repeatability within the same numerical execution profile, including batch shapes, cache mode and kernels. Keep random streams isolated and scheduling-independent, and preserve explicit-seed choice derivation as `seed + choice_index`. A profile change can change logits and therefore text despite the same random seed; stronger cross-profile deterministic execution was not selected. This does not waive stream isolation, same-profile repeatability, or streaming/non-streaming conformance.
