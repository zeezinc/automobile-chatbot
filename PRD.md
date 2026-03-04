# Product Requirements Document (PRD)

## Guardrails & DON’Ts
1. Do not invent components/wires absent in graph.
2. Do not answer from model priors when graph lookup fails.
3. Always cite component keys/pages used for answer derivation.
4. Return `NOT_FOUND`/`INSUFFICIENT_GRAPH_DATA` for missing graph evidence.
5. Block unsupported asks (e.g., safety-critical diagnosis claims) with safe refusal template.
# Product Requirements Document (PRD): Confidence-Aware Knowledge Graph Querying

## Overview
This update introduces confidence-aware behavior across parsing and query response flows so consumers can assess extraction quality and uncertainty.

## Goals
- Surface parser confidence at graph element level (nodes and edges).
- Expose confidence insights in query responses.
- Add guardrails for low-confidence outputs.
- Track low-confidence rates for observability.
- Allow clients to opt in to confidence details.

## Requirements

### 1) Parser-stage confidence fields for nodes and edges
- The parser **must** output a `confidence` field for every extracted node.
- The parser **must** output a `confidence` field for every extracted edge.
- Confidence values should be normalized floating-point values in the range `[0.0, 1.0]`.
- Missing confidence values should be treated as `null` and counted in quality metrics.

### 2) Query response payload includes confidence summary and uncertainty note
When confidence is requested (see optional parameter below), query responses should include:
- `confidence_summary` object containing:
  - `node_confidence_avg`
  - `edge_confidence_avg`
  - `overall_confidence`
  - `low_confidence_node_ratio`
  - `low_confidence_edge_ratio`
- `uncertainty_note` string explaining whether results are reliable or require caution.

### 3) Threshold-based warnings
- Introduce configurable threshold(s) for warnings.
- Default threshold: `0.65`.
- If `overall_confidence < 0.65`, set `uncertainty_note` to include: **"verify manually"**.
- If node/edge averages are below threshold, include per-dimension warnings in `uncertainty_note`.

### 4) Metrics for low-confidence node/edge ratio
System metrics must be emitted for:
- `low_confidence_node_ratio` = nodes below threshold / total nodes.
- `low_confidence_edge_ratio` = edges below threshold / total edges.
- Ratios should be produced per query and also aggregated over time for dashboards/alerts.

### 5) Optional query parameter `include_confidence=true`
- Add optional query parameter: `include_confidence`.
- Default behavior: `false` (response omits confidence payload extensions).
- If `include_confidence=true`, include:
  - `confidence_summary`
  - `uncertainty_note`
  - Any threshold-based warning annotations.

## Example Response (when `include_confidence=true`)
```json
{
  "query": "Which sensors are likely failing?",
  "results": [
    { "entity": "O2 Sensor", "status": "degraded" }
  ],
  "confidence_summary": {
    "node_confidence_avg": 0.71,
    "edge_confidence_avg": 0.62,
    "overall_confidence": 0.66,
    "low_confidence_node_ratio": 0.18,
    "low_confidence_edge_ratio": 0.41
  },
  "uncertainty_note": "Some relationship confidence is low; verify manually before acting."
}
```

## Acceptance Criteria
- Parser emits node and edge confidence values in `[0.0, 1.0]`.
- API supports `include_confidence=true` and conditionally adds confidence fields.
- Warning text includes "verify manually" when `overall_confidence < 0.65`.
- Node and edge low-confidence ratios are computed and exposed in metrics.
- Backward compatibility maintained when `include_confidence` is omitted.
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
## Schema

### `component_key` generation rule

`component_key` **must** be generated deterministically using the following normalization pipeline so the same visual label always maps to the same graph node:

1. Start with `label_raw`.
2. Apply canonical normalization:
   - convert to uppercase,
   - replace non-alphanumeric characters with single underscores,
   - collapse repeated underscores,
   - trim leading/trailing underscores.
3. Derive a `page` hint token:
   - normalize `page` with the same uppercase alphanumeric rule,
   - prefix with `P` if the page is numeric-only,
   - use `UNKNOWN_PAGE` when page cannot be resolved.
4. Compose key as: `<LABEL_NORMALIZED>__<PAGE_HINT>`.
5. If collision remains within the same payload, append stable ordinal suffix `__N` (1-based) in detected order.

Example: `"IGN relay #1"` on page `12` → `IGN_RELAY_1__P12`.

### Component schema (required fields)

Every component object inserted into the graph **must include** all of the fields below:

- `component_key` (string): deterministic unique key generated by the rule above.
- `label_raw` (string): original extracted label text.
- `label_normalized` (string): normalized label token used for matching.
- `type` (enum): component category (see **Type enumeration**).
- `page` (string): source page identifier where the component appears.
- `bbox` (object): bounding box in page coordinates.
- `confidence` (number): extraction confidence in `[0.0, 1.0]`.

`bbox` structure:

- `x_min` (number)
- `y_min` (number)
- `x_max` (number)
- `y_max` (number)

Validation constraints:

- `x_min < x_max`
- `y_min < y_max`

### Connection schema (required fields)

Every connection object inserted into the graph **must include** all of the fields below:

- `from_component_key` (string): origin component key.
- `to_component_key` (string): destination component key.
- `wire_color_raw` (string): original extracted wire color notation.
- `wire_color_normalized` (string): normalized wire color token.
- `direction` (enum): direction of current/signal flow (see **Direction enumeration**).
- `page` (string): source page identifier where the wire segment is observed.
- `cross_page_ref` (string|null): cross-page pointer when connection spans pages.
- `confidence` (number): extraction confidence in `[0.0, 1.0]`.

Validation constraints:

- `from_component_key != to_component_key` unless explicitly marked as loop/test connection.
- `from_component_key` and `to_component_key` must resolve to known components in the payload.

### Direction enumeration

Allowed `direction` values:

- `incoming`
- `outgoing`
- `bidirectional`
- `unknown`

### Type enumeration

Allowed `type` values (initial canonical set, extensible through versioned schema updates):

- `PowerSource`
- `Ground`
- `Fuse`
- `FusibleLink`
- `Relay`
- `Switch`
- `Connector`
- `Splice`
- `Junction`
- `ControlUnit`
- `Sensor`
- `Actuator`
- `Lamp`
- `Motor`
- `Resistor`
- `Diode`
- `Capacitor`
- `Transistor`
- `Terminal`
- `Bus`
- `TestPoint`
- `Unknown`

### Payload validation and repair rule

All payloads **must be validated before graph insertion**.

- If a record fails required-field checks, enum checks, range checks, or key-resolution checks, the ingest layer must attempt deterministic repair first (e.g., normalize enum casing, regenerate `component_key`, clamp confidence to `[0,1]` when recoverable, coerce null-like values).
- If required fields are missing or repair cannot make the record schema-valid without guesswork, the payload (or invalid records in partial-ingest mode) must be rejected and logged with explicit validation errors.
- No invalid component or connection may be inserted into the graph.
