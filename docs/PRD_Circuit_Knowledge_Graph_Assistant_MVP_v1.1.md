# Product Requirements Document (PRD)
## Circuit Knowledge Graph Assistant (CKGA) – MVP/POC v1.1

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
      "component_key": "RELAY_R1",
      "label_raw": "Relay R1",
      "label_normalized": "RELAY R1",
      "type": "Relay",
      "bbox": [120, 340, 180, 380],
      "confidence": 0.92
    }
  ],
  "connections": [
    {
      "from_component_key": "BATTERY_MAIN",
      "to_component_key": "FUSE_F1",
      "wire_color_raw": "Red",
      "wire_color_normalized": "RED",
      "direction": "outgoing",
      "cross_page_ref": null,
      "confidence": 0.89
    }
  ]
}
```

#### Normalization rules
- `component_key`: deterministic uppercase key, e.g. `TYPE_LABELTOKEN` format.
- `label_normalized`: uppercase, trimmed, condensed spacing.
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
- `pages` (set/list)
- `confidence`

#### Edge attributes
- `source`
- `target`
- `wire_color_normalized`
- `direction`
- `page`
- `cross_page_ref`
- `confidence`

#### Special processing
- Identify battery nodes (`type == PowerSource` OR label contains battery token).
- Compute flow depth from battery node(s) via DFS/BFS.
- Track unresolved cross-page edges for second pass resolution.

---

### FR4 – Query Agent
**Endpoint**: `POST /query`

**Input example**:
```json
{ "question": "How many and which incoming wires are there for Relay R1?" }
```

#### Query planning contract
LLM must convert user question to DSL JSON:
```json
{
  "intent": "COUNT_AND_LIST_INCOMING",
  "component": "Relay R1"
}
```

#### Supported intents
- `GET_INCOMING`
- `GET_OUTGOING`
- `COUNT_INCOMING`
- `COUNT_OUTGOING`
- `COUNT_AND_LIST_INCOMING`
- `COUNT_AND_LIST_OUTGOING`
- `TRACE_FROM_BATTERY`
- `FIND_PATH`

#### Execution behavior
1. Resolve component text to `component_key` (exact → normalized → fuzzy).
2. If ambiguous, return clarification payload with top candidates.
3. Execute deterministic graph query.
4. Return structured answer + evidence + confidence warning if needed.

**Response example**:
```json
{
  "answer": "Relay R1 has 2 incoming wires: RED from Fuse F1 and BLK from Connector C2.",
  "intent": "COUNT_AND_LIST_INCOMING",
  "component_key": "RELAY_R1",
  "incoming_count": 2,
  "incoming_wires": [
    {"from": "FUSE_F1", "wire_color": "RED", "page": 2},
    {"from": "CONNECTOR_C2", "wire_color": "BLK", "page": 3}
  ],
  "evidence": {
    "nodes_used": ["RELAY_R1", "FUSE_F1", "CONNECTOR_C2"],
    "edges_used": ["FUSE_F1->RELAY_R1", "CONNECTOR_C2->RELAY_R1"],
    "pages_used": [2, 3]
  },
  "confidence_summary": {
    "min_edge_confidence": 0.71,
    "warning": "Some extracted links are low confidence; verify on source diagram."
  }
}
```

---

### FR5 – Metrics Endpoint
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
  "low_confidence_edge_ratio": 0.14
}
```

---

## 5) Non-Functional Requirements
- Query latency target: `< 3s`.
- Parsing latency target: `< 5s/page`.
- Strict JSON validation for parser and query planner output.
- Graceful failure on invalid JSON or unresolved component.
- Structured logs for all LLM calls (without leaking secrets).

---

## 6) Guardrails and DON’Ts

### Must-do guardrails
1. **Graph-grounded answers only**: all answers must be derived from graph queries.
2. **Evidence required**: include node/edge/page evidence in response payload.
3. **Ambiguity flow**: if multiple component matches, request clarification.
4. **Confidence warning**: show warning when confidence threshold falls below configured minimum.
5. **Error taxonomy**: return explicit codes: `NOT_FOUND`, `AMBIGUOUS_COMPONENT`, `INSUFFICIENT_GRAPH_DATA`, `INVALID_QUERY_PLAN`.

### DON’Ts
- Don’t invent components, wires, directions, pages, or paths absent in graph.
- Don’t answer from prior automotive knowledge when graph lacks data.
- Don’t silently choose one of multiple component matches.
- Don’t provide safety-critical diagnostic claims (e.g., “this car is safe to drive”) from incomplete graph context.
- Don’t expose API keys, raw secrets, or sensitive logs.

---

## 7) Tech Stack
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

## 8) Suggested Repository Structure
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
│   └── PRD_Circuit_Knowledge_Graph_Assistant_MVP_v1.1.md
├── README.md
└── docker-compose.yml
```

---

## 9) Implementation Notes
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

## 10) Metrics to Track
- Components detected per page
- Wires detected per page
- Unresolved cross-page refs
- Parse latency (avg/p95)
- Query latency (avg/p95)
- Graph depth from battery
- Low-confidence node/edge ratio
- Query success/failure by error code

---

## 11) MVP Evaluation Cases
- “List outgoing wires from Battery.”
- “Which wires go into Relay R1?”
- “How many incoming wires are there for component A?”
- “Trace power flow from Battery to Headlight.”
- “Find path between Fuse F1 and ECU.”
- Ambiguous case: “Show incoming wires to Relay.” → should request clarification.

---

## 12) POC Success Criteria
- ≥ 75% component extraction accuracy on validation set.
- Incoming/outgoing wire query correctness on golden fixtures.
- Flow trace from battery works on representative diagrams.
- Query p95 latency < 3 seconds under target load.
- No hallucinated answers in red-team test prompts.

---

## 13) Risk Areas and Mitigations
1. **LLM invalid JSON** → schema validation + retry + fallback error.
2. **Over/under-detected wires** → confidence scoring + manual review mode.
3. **Cross-page complexity** → second-pass resolver + unresolved ref reporting.
4. **Label ambiguity** → deterministic normalization + clarification workflow.

---

## 14) Future Upgrades (Phase 2+)
- Replace NetworkX with Neo4j.
- Cypher generation and graph analytics.
- Visual graph explorer UI.
- Persistent storage and versioned graphs.
- Fine-tuned extraction model for improved precision.

---

## 15) How to Use This PRD in Antigravity
1. Create a project in Antigravity and upload this PRD.
2. Ask Antigravity to generate epics from sections 4–6 and 10–12.
3. Generate implementation tasks in this order:
   - Backend skeleton + schemas
   - `/upload` parser flow
   - Graph service + query intents
   - `/query` execution + guardrails
   - Metrics/logging
   - Frontend chat/upload
4. Use section 12 as Definition of Done gates.
5. Run weekly PRD drift review (implementation vs PRD contracts).

