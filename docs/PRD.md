# Product Requirements Document (PRD)
## Circuit Knowledge Graph Assistant (CKGA) – MVP/POC v1.1 (Consolidated)

## 1) Objective
Build a fully working MVP/POC that can ingest automotive circuit diagram PDFs, extract graph entities and relationships, and support chatbot-style question answering grounded in the extracted graph.

Core target capability:
- Identify **components**.
- Identify **incoming/outgoing branches (wires)** for each component.
- Identify **relationships and power-flow paths** between components.
- Answer questions like:
  - "How many incoming wires are there for component A?"
  - "Which incoming wires go into Relay R1?"
  - "What are outgoing branches from Fuse F1?"
  - "Find path between Fuse F1 and ECU."

---

## 2) Scope
### In Scope (MVP)
- Upload automotive circuit PDF(s).
- Parse pages to structured entities/relations with LLM vision/text.
- Build in-memory directed graph (NetworkX `DiGraph`).
- Query graph via API and chat UI.
- Track metrics and structured logs.
- Add strict validation and guardrails to reduce hallucinations.
- Apply confidence-aware querying and response formatting.

### Out of Scope (Phase 1)
- Multi-user auth/roles.
- Persistent graph DB (Neo4j production setup).
- High-precision OCR model tuning/fine-tuning.
- Full diagnostic/safety decision engine.

---

## 3) Architecture
Frontend (React Chat + Upload)
→ FastAPI Backend
→ Parser Agent (Vision/Text + schema-constrained extraction)
→ Validation & Normalization Layer
→ Graph Builder (`networkx.DiGraph`)
→ Query Planner + Query Executor
→ Response Formatter (evidence + confidence + warnings)

---

## 4) Functional Requirements

### FR1 – Upload and Parse Circuit Diagram
**Endpoint**: `POST /upload`

**Input**:
- Multipart PDF file upload.

**Behavior**:
1. Validate file type and size.
2. Convert each PDF page to image (`PyMuPDF`).
3. Send page images to parser agent.
4. Validate and normalize parser output against strict schema.
5. Build/update in-memory graph.
6. Return parse summary.

**Response example**:
```json
{
  "document_id": "doc_2026_01_15_001",
  "pages_processed": 8,
  "total_nodes": 42,
  "total_edges": 57,
  "battery_nodes": ["BATTERY_MAIN"],
  "unresolved_cross_page_refs": 3,
  "parse_time_ms": 1840
}
```

---

### FR2 – Diagram Parsing Agent
**Input**: page image + optional OCR text

**Output**: strict JSON only.

#### Canonical schema (required)
```json
{
  "page": 1,
  "components": [
    {
      "component_key": "RELAY_R1__P1",
      "label_raw": "Relay R1",
      "label_normalized": "RELAY_R1",
      "type": "Relay",
      "page": "1",
      "bbox": {
        "x_min": 120,
        "y_min": 340,
        "x_max": 180,
        "y_max": 380
      },
      "confidence": 0.92
    }
  ],
  "connections": [
    {
      "from_component_key": "BATTERY_MAIN__P1",
      "to_component_key": "FUSE_F1__P1",
      "wire_color_raw": "Red",
      "wire_color_normalized": "RED",
      "direction": "outgoing",
      "page": "1",
      "cross_page_ref": null,
      "confidence": 0.89
    }
  ]
}
```

#### Normalization and key rules
- `component_key` must be deterministic from `label_raw` and page hint:
  1. Uppercase.
  2. Replace non-alphanumeric with single `_`.
  3. Collapse repeated `_` and trim leading/trailing `_`.
  4. Normalize page token using same rule, prefix numeric-only page with `P`, use `UNKNOWN_PAGE` if unresolved.
  5. Compose as `<LABEL_NORMALIZED>__<PAGE_HINT>`.
  6. If collision remains in the same payload, append stable suffix `__N` (1-based).
- `label_normalized`: uppercase tokenized canonical label.
- `wire_color_normalized`: uppercase enum-like string (`RED`, `BLK`, `YEL`, etc.).
- `direction` enum: `incoming | outgoing | bidirectional | unknown`.

#### Parsing prompt constraints
- Detect rectangular/labeled components.
- Extract labels exactly as seen in `label_raw`.
- Infer normalized labels and keys using deterministic rules.
- Detect wire color and direction where visible.
- Detect cross-page references/connectors.
- Output JSON only (no prose).

#### Model options
- GPT-4o (recommended for multimodal stability)
- Claude 3 Opus
- Gemini 1.5 Pro

---

### FR3 – Graph Builder
Use `NetworkX DiGraph`.

#### Node attributes
- `component_key` (primary ID)
- `label_raw`
- `label_normalized`
- `type`
- `page`
- `bbox`
- `confidence`

#### Edge attributes
- `from_component_key`
- `to_component_key`
- `wire_color_raw`
- `wire_color_normalized`
- `direction`
- `page`
- `cross_page_ref`
- `confidence`

#### Special processing
- Identify battery nodes (`type == PowerSource` OR label contains battery token).
- Compute flow depth from battery node(s) via DFS/BFS.
- Track unresolved cross-page edges for second-pass resolution.

---

### FR4 – Query Agent
**Endpoint**: `POST /query`

**Input example**:
```json
{ "question": "How many and which incoming wires are there for Relay R1?" }
```

#### Query planning contract
LLM must convert user question to DSL JSON only:
```json
{
  "intent": "COUNT_AND_LIST_INCOMING",
  "component": "Relay R1"
}
```

Required fields:
- `intent` (string): one supported backend intent.
- `component` (string): raw component reference extracted from user input.

Optional fields:
- `time_range` (object)
- `filters` (object)
- `metadata` (object)

#### Supported intents
- `GET_INCOMING`
- `GET_OUTGOING`
- `COUNT_INCOMING`
- `COUNT_OUTGOING`
- `COUNT_AND_LIST_INCOMING`
- `COUNT_AND_LIST_OUTGOING`
- `TRACE_FROM_BATTERY`
- `FIND_PATH`

#### Component-resolution behavior
1. Resolve component text to canonical key (exact → normalized → fuzzy).
2. Score candidates with deterministic confidence scoring.
3. If multiple plausible matches exceed ambiguity threshold, return ambiguity response.
4. If no candidate passes minimum threshold, skip data retrieval and return `UNKNOWN_COMPONENT`.

Determinism requirements:
- Same input + same registry snapshot => same output.
- Candidate sorting: descending confidence, then lexical name.
- Include top-3 nearest candidates when available.

#### Ambiguity response format
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

#### LLM output and validation requirements
1. Parse raw LLM output as JSON.
2. Validate query JSON with Pydantic models before backend execution.
3. Reject invalid payloads with structured validation error response.
4. Execute backend logic only on validated payloads.

Validation failure response should include:
- `status: "INVALID_QUERY_PAYLOAD"`
- `errors`: schema violations
- `message`: user-safe retry prompt

#### Query execution response requirements
- Graph-grounded answer only.
- Include evidence (`nodes_used`, `edges_used`, `pages_used`).
- Include confidence payload only when `include_confidence=true`.

---

### FR5 – Confidence-Aware Querying
#### Parser-stage confidence
- Parser must output `confidence` for every node and edge.
- Confidence values are normalized floats in `[0.0, 1.0]`.
- Missing confidence values treated as `null` and counted in quality metrics.

#### Optional query parameter
- `include_confidence` (default `false`).
- If `true`, response must include:
  - `confidence_summary`
  - `uncertainty_note`
  - threshold-based warnings

#### Confidence summary fields
- `node_confidence_avg`
- `edge_confidence_avg`
- `overall_confidence`
- `low_confidence_node_ratio`
- `low_confidence_edge_ratio`

#### Threshold-based warnings
- Configurable threshold; default `0.65`.
- If `overall_confidence < 0.65`, `uncertainty_note` must include: **"verify manually"**.
- If node/edge averages are below threshold, include per-dimension warnings.

---

### FR6 – Metrics Endpoint
**Endpoint**: `GET /metrics`

**Returns**:
```json
{
  "total_nodes": 42,
  "total_edges": 57,
  "total_queries": 12,
  "parse_time_ms_avg": 1840,
  "query_latency_ms_p95": 320,
  "unresolved_cross_page_refs": 3,
  "low_confidence_node_ratio": 0.11,
  "low_confidence_edge_ratio": 0.14
}
```

System metrics must be emitted per query and aggregated over time for dashboards/alerts.

---

## 5) Data Schema and Validation

### Component schema (required)
- `component_key` (string)
- `label_raw` (string)
- `label_normalized` (string)
- `type` (enum)
- `page` (string)
- `bbox` (object with `x_min`, `y_min`, `x_max`, `y_max`)
- `confidence` (number in `[0.0, 1.0]`)

Validation constraints:
- `x_min < x_max`
- `y_min < y_max`

### Connection schema (required)
- `from_component_key` (string)
- `to_component_key` (string)
- `wire_color_raw` (string)
- `wire_color_normalized` (string)
- `direction` (enum)
- `page` (string)
- `cross_page_ref` (string|null)
- `confidence` (number in `[0.0, 1.0]`)

Validation constraints:
- `from_component_key != to_component_key` unless explicitly marked loop/test.
- Connection endpoints must resolve to known components.

### Direction enumeration
- `incoming`
- `outgoing`
- `bidirectional`
- `unknown`

### Type enumeration
- `PowerSource`, `Ground`, `Fuse`, `FusibleLink`, `Relay`, `Switch`, `Connector`, `Splice`, `Junction`, `ControlUnit`, `Sensor`, `Actuator`, `Lamp`, `Motor`, `Resistor`, `Diode`, `Capacitor`, `Transistor`, `Terminal`, `Bus`, `TestPoint`, `Unknown`

### Payload validation and repair rule
All payloads must be validated before graph insertion.
- Deterministic repair is allowed for recoverable issues (enum casing, key regeneration, clamping confidence, null-like coercion).
- If required fields are missing or repair requires guesswork, reject payload (or invalid records in partial-ingest mode) and log explicit errors.
- No invalid component or connection may be inserted.

---

## 6) Guardrails and DON’Ts

### Must-do guardrails
1. Graph-grounded answers only.
2. Evidence required: cite component keys/pages/edges used.
3. Ambiguity flow: never silently pick among multiple plausible components.
4. Confidence warning on low-confidence outputs.
5. Error taxonomy with explicit statuses: `NOT_FOUND`, `UNKNOWN_COMPONENT`, `AMBIGUOUS_COMPONENT`, `INSUFFICIENT_GRAPH_DATA`, `INVALID_QUERY_PLAN`, `INVALID_QUERY_PAYLOAD`.

### DON’Ts
1. Do not invent components/wires absent in graph.
2. Do not answer from model priors when graph lookup fails.
3. Do not omit citations for derivation evidence.
4. Return `NOT_FOUND`/`INSUFFICIENT_GRAPH_DATA` when graph evidence is missing.
5. Block unsupported safety-critical asks with safe refusal template.
6. Don’t expose API keys, raw secrets, or sensitive logs.

---

## 7) Non-Functional Requirements
- Query latency target: `< 3s`.
- Parsing latency target: `< 5s/page`.
- Strict JSON validation for parser and query planner output.
- Graceful failure on invalid JSON or unresolved component.
- Structured logs for all LLM calls (without leaking secrets).

---

## 8) Tech Stack
### Backend
- FastAPI
- Uvicorn
- NetworkX
- PyMuPDF
- Pydantic
- Python logging / Loguru (optional)
- OpenAI SDK
- Prometheus (optional)

### Frontend
- React (Vite)
- Axios
- Tailwind CSS

### LLM
- GPT-4o for vision extraction
- GPT-4.1 (or equivalent) for structured query planning/formatting

---

## 9) Suggested Repository Structure
```text
circuit-kg-assistant/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── models/schemas.py
│   │   ├── agents/
│   │   │   ├── parser_agent.py
│   │   │   ├── graph_agent.py
│   │   │   └── query_agent.py
│   │   ├── services/
│   │   │   ├── pdf_service.py
│   │   │   └── graph_service.py
│   │   ├── routes/
│   │   │   ├── upload.py
│   │   │   ├── query.py
│   │   │   └── metrics.py
│   │   ├── utils/
│   │   │   ├── logger.py
│   │   │   └── metrics.py
│   │   └── graph/graph_store.py
│   ├── requirements.txt
│   └── .env
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   │   ├── ChatBox.jsx
│   │   │   ├── Message.jsx
│   │   │   └── Upload.jsx
│   │   └── api.js
│   └── package.json
├── docs/
│   └── PRD.md
├── README.md
└── docker-compose.yml
```

---

## 10) Implementation Notes
### Parser flow
1. PDF page → image.
2. Image → LLM with strict schema instruction.
3. Validate with Pydantic.
4. Retry once on invalid JSON.
5. Drop/flag low-confidence ambiguous edges.

### Graph service API (minimum)
- `add_component(node)`
- `add_connection(edge)`
- `get_incoming(component_key)`
- `get_outgoing(component_key)`
- `count_incoming(component_key)`
- `count_outgoing(component_key)`
- `trace_from_battery()`
- `find_path(source, target)`

### Query planner prompt (strict)
> You are a circuit graph query planner. Convert the user question into one supported intent JSON. Do not answer the user directly. Output strict JSON only.

---

## 11) Metrics to Track
- Components detected per page
- Wires detected per page
- Unresolved cross-page refs
- Parse latency (avg/p95)
- Query latency (avg/p95)
- Graph depth from battery
- Low-confidence node/edge ratio
- Query success/failure by error code

---

## 12) MVP Evaluation Cases
- “List outgoing wires from Battery.”
- “Which wires go into Relay R1?”
- “How many incoming wires are there for component A?”
- “Trace power flow from Battery to Headlight.”
- “Find path between Fuse F1 and ECU.”
- Ambiguous case: “Show incoming wires to Relay.” → should request clarification.

---

## 13) POC Success Criteria
- ≥ 75% component extraction accuracy on validation set.
- Incoming/outgoing wire query correctness on golden fixtures.
- Flow trace from battery works on representative diagrams.
- Query p95 latency < 3 seconds under target load.
- No hallucinated answers in red-team test prompts.

---

## 14) Risk Areas and Mitigations
1. **LLM invalid JSON** → schema validation + retry + fallback error.
2. **Over/under-detected wires** → confidence scoring + manual review mode.
3. **Cross-page complexity** → second-pass resolver + unresolved ref reporting.
4. **Label ambiguity** → deterministic normalization + clarification workflow.

---

## 15) Future Upgrades (Phase 2+)
- Replace NetworkX with Neo4j.
- Cypher generation and graph analytics.
- Visual graph explorer UI.
- Persistent storage and versioned graphs.
- Fine-tuned extraction model for improved precision.

---

## 16) How to Use This PRD in Antigravity
1. Create a project in Antigravity and upload this PRD.
2. Ask Antigravity to generate epics from sections 4–7 and 11–13.
3. Generate implementation tasks in this order:
   - Backend skeleton + schemas
   - `/upload` parser flow
   - Graph service + query intents
   - `/query` execution + guardrails
   - Metrics/logging
   - Frontend chat/upload
4. Use section 13 as Definition of Done gates.
5. Run weekly PRD drift review (implementation vs PRD contracts).
