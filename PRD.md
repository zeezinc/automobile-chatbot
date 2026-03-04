# Product Requirements Document (PRD)

## Guardrails & DON’Ts
1. Do not invent components/wires absent in graph.
2. Do not answer from model priors when graph lookup fails.
3. Always cite component keys/pages used for answer derivation.
4. Return `NOT_FOUND`/`INSUFFICIENT_GRAPH_DATA` for missing graph evidence.
5. Block unsupported asks (e.g., safety-critical diagnosis claims) with safe refusal template.
