---
status: accepted
---

# Maintain a pinned Limite patch series

The user accepted maintaining an exact upstream llama.cpp revision plus an auditable, ordered Limite patch series in an isolated Vulkan build/package path. Reuse the engine's existing model, server, and scheduling machinery where it satisfies Limite's semantics; architecture support, BF16 arithmetic, cache integration, and compatibility gaps remain explicit patch responsibilities. This avoids both waiting for upstream Limite support and allowing an unrestricted long-lived engine fork to obscure the changes required by this model.

`296538adea19ff397100bf8e7d71f302d2d2bb35` remains a candidate baseline, not a qualified dependency. The user accepted a checksum-verified immutable upstream source input, reusable from an offline build cache, plus local ordered patches instead of a submodule or fully vendored engine. Fetch only during explicit source preparation/builds; ordinary serving uses a pinned native binary, never an unpinned checkout or runtime compilation. Record source, patches, toolchain, and artifact provenance together.

The first qualification targets are native Windows 11 x86-64 and Ubuntu 24.04 x86-64. The user accepted platform-specific Python wheels in a separately locked Vulkan package, containing the pinned worker and compiled shaders; end users need a compatible driver/loader, not the Vulkan SDK or runtime compilation. Publish the exact qualified build/driver fingerprints instead of claiming broader platform support from package availability.

The brief's compiler/SDK versions remain build candidates until exercised. Patch/build tooling, wheel ABI and binary dependency audits, and observed toolchain/platform fingerprints still need specification and qualification. Cache adaptation is recorded in ADR-0007; worker token ownership and control are recorded in ADR-0004. No native implementation or release artifact exists yet.
