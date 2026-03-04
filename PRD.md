# Product Requirements Document (PRD)

## Query Contract

### 1) JSON DSL for query intent
The assistant MUST express user query intent using a JSON-only DSL with the following baseline schema:

```json
{
  "intent": "GET_INCOMING",
  "component": "Relay R1"
}
```

Required fields:
- `intent` (string): one of the supported backend intents.
- `component` (string): raw component reference extracted from user input.

Optional fields (if needed by intent-specific handlers):
- `time_range` (object)
- `filters` (object)
- `metadata` (object)

### 2) Component-resolution step
Before backend execution, the system MUST resolve `component` against a canonical component registry using:
1. Fuzzy matching against canonical names and aliases.
2. Confidence scoring for each candidate.
3. Confirmation flow if multiple plausible matches exceed the ambiguity threshold.

Resolution outcomes:
- **Single high-confidence match**: proceed automatically.
- **Multiple plausible matches**: return ambiguity guardrail response (see section 4).
- **No plausible match**: follow deterministic unknown-component behavior (see section 3).

### 3) Deterministic backend behavior for unknown components
If no candidate passes the minimum confidence threshold, the backend MUST:
1. Skip execution of data retrieval actions.
2. Return a structured `UNKNOWN_COMPONENT` status.
3. Include a user-safe message asking for clarification.
4. Include up to top-3 nearest candidates (if available) with confidence scores.

This behavior MUST be deterministic:
- Same input + same registry snapshot => same output status and candidate ordering.
- Candidate sorting order: descending confidence, then lexical component name as tie-breaker.

### 4) Guardrail response format for ambiguity
For ambiguous component resolution, the system MUST return the following JSON shape:

```json
{
  "status": "AMBIGUOUS_COMPONENT",
  "query": {
    "intent": "GET_INCOMING",
    "component": "Relay"
  },
  "candidates": [
    { "name": "Relay R1", "confidence": 0.91 },
    { "name": "Relay R2", "confidence": 0.87 },
    { "name": "Relay Main", "confidence": 0.79 }
  ],
  "message": "I found multiple matching components. Please confirm one of the candidates.",
  "requires_confirmation": true
}
```

Requirements:
- Include exactly the top-3 candidates when at least 3 exist; otherwise include all available.
- `confidence` is a normalized float in `[0.0, 1.0]`.
- Response MUST be machine-parseable and stable in key naming.

### 5) LLM output and validation requirements
The LLM MUST output **JSON only** (no markdown, prose, or code fences).

Execution pipeline requirement:
1. Parse raw LLM output as JSON.
2. Validate the JSON with Pydantic models before any backend execution.
3. Reject invalid payloads with a structured validation error response.
4. Execute backend logic only on validated payloads.

Validation failure response SHOULD include:
- `status: "INVALID_QUERY_PAYLOAD"`
- `errors`: array of schema violations
- `message`: user-safe retry prompt
