---
status: accepted
---

# Python owns canonical text semantics

The user accepted a single Python/Hugging Face text module owning canonical prompt rendering, prompt encoding, output decoding, and text-stop interpretation. The native worker consumes and emits token IDs and owns numerical execution, sampling, generation state, EOS, and explicit stop-token handling. This preserves the published vocabulary and raw reasoning delimiters instead of encoding with Hugging Face and trusting an independently interpreted GGUF vocabulary to decode the result.

The user accepted a raw token-ID event channel independent of native text filtering and UTF-8 buffering, with bounded run-ahead/backpressure and an acknowledged terminal prefix and cleanup. Python alone matches text stops. Final usage must match the reference-compatible accepted prefix, and later native computation must be separately observable rather than silently included in the response or reused as accepted state. This avoids a separate HTTP acknowledgement after every token without allowing unbounded work past a text-stop decision.

The exact run-ahead bound, control-message sequencing, transport schema, and cancellation/terminal acknowledgement conditions require a concrete interface and verification. This acceptance is not a performance measurement or a claim that the existing native text-stream path already satisfies the token contract.
