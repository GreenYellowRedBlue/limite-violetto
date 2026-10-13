---
status: accepted
---

# Preserve one evaluation contract across engines

The user accepted retaining one scoring and measurement contract while introducing engine-specific launch/provenance adapters and isolated runtime dependencies for CUDA and Vulkan. Keep the existing CUDA default and historical manifests intact, require an explicit cross-engine comparison mode, and record the observed engine, source/model artifact, rendering, and numerical identity. A separate Vulkan evaluator or a spoofed vLLM version would either duplicate measurement policy or mislabel the execution that produced a result.

The user also accepted a dedicated strict cross-engine qualification policy, separate from the existing generic comparison verdict. Require the same verified source snapshot and lossless derivation, actual text-asset identities, frozen resolved benchmark/rendered inputs, scorer, sampling, budgets, stops, and decode policy. Runtime-artifact byte hashes remain distinct; only explicitly declared engine/build/layout differences are allowed. New qualification freezes the actually resolved GSM8K inputs without retroactively inventing that evidence in historical manifests.

Portable launch behavior, dependency/lockfile organization and the qualification manifest/interface remain implementation obligations. ADR-0008 settles the finite-corpus confidence construction, randomized paired-seed policy, single-look rule and severe-taskset release gate; actual corpus, dependence, coverage and execution records still require qualification. The evaluator currently admits an older Violetto source snapshot and a different CUDA engine environment; accepting this architecture does not rewrite those inputs or their historical outputs.
