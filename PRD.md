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
