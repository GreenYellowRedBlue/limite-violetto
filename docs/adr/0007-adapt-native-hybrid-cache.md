---
status: accepted
---

# Adapt the native hybrid cache before replacing its allocator

The user accepted adapting llama.cpp's existing global/sliding cache and tensor allocator machinery before introducing a separate pager. Preserve complete global history, exact local visibility, value-embedding-adjusted V, absolute positions, chunk-safe writes, and safe reclamation. This reuses an existing split-state implementation instead of treating the brief's 64-token pages and 512 MiB slabs as inherited requirements or prematurely committing to an independent allocator.

Prefix reuse stays disabled until complete-state parity passes; partial-only snapshots are insufficient for Limite's global layers. Page/slab constants remain candidates, and the full-context and declared serving targets are unchanged. Actual tensors, allocation limits, workspace, admission, cancellation and restore behavior still require target-platform qualification; generic tensor-list splitting does not prove that an oversized individual tensor is supported.
