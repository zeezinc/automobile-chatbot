# Product Requirements Document (PRD)

## MVP acceptance tests

| Test category | Fixture / test ID | Input / setup | Expected result | Pass / fail threshold | Manual verification protocol |
| --- | --- | --- | --- | --- | --- |
| Graph extraction (happy path) | `fixture_sedan_brake_system_v1` | Parse canonical sedan brake-system PDF fixture. | Node count = `42`; edge count = `57`; required key paths present: `Vehicle/BrakeSystem/MasterCylinder`, `Vehicle/BrakeSystem/ABSModule`, `Vehicle/BrakeSystem/FrontLeftCaliper`. | **Pass** if all expected counts and all key paths match exactly. **Fail** if any mismatch. | 1) Run ingestion against fixture. 2) Export graph summary. 3) Confirm counts. 4) Resolve each key path in graph explorer and capture screenshot/log snippet. |
| Graph extraction (variant fixture) | `fixture_ev_thermal_loop_v1` | Parse EV thermal-loop PDF fixture. | Node count = `35`; edge count = `46`; required key paths present: `Vehicle/ThermalSystem/CoolantPump`, `Vehicle/ThermalSystem/BatteryChiller`, `Vehicle/ThermalSystem/Radiator`. | **Pass** if counts are within exact expected values and all key paths resolve. **Fail** otherwise. | 1) Run ingestion. 2) Validate summary JSON fields `node_count`, `edge_count`, `key_paths_found`. 3) Spot-check one path in UI and one via API. |
| Query answer contract | `query_brake_hydraulic_path` | Query: “What is the hydraulic flow path from brake pedal to front-left caliper?” | Exact answer structure: `{ "answer": "<string>", "evidence_paths": ["Vehicle/..."], "node_ids": ["<id>"], "confidence": <0.0-1.0>, "citations": [{"source":"<fixture>","page":<int>}]} ` with non-empty `answer`, >=1 `evidence_paths`, >=2 `node_ids`, `confidence >= 0.8`. | **Pass** if response matches schema and all required fields/constraints exactly. **Fail** on schema drift, missing field, or confidence below threshold. | 1) Execute query through API test harness. 2) Validate JSON with contract schema. 3) Manually inspect that cited page mentions each evidence path endpoint. |
| Query answer contract | `query_abs_component_role` | Query: “What is the role of the ABS module in this system?” | Exact answer structure: `{ "answer": "<string>", "component": "ABSModule", "relationships": [{"type":"<edge_type>","target":"<component>"}], "confidence": <0.0-1.0>, "citations": [{"source":"<fixture>","page":<int>}]} ` with >=1 relationship and confidence >= 0.8. | **Pass** if JSON structure and constraints are met and `component` is exactly `ABSModule`. **Fail** otherwise. | 1) Run query. 2) Compare response against golden JSON schema. 3) Manually verify one listed relationship in rendered graph. |
| Error path | `error_invalid_pdf` | Upload malformed/non-PDF binary with `.pdf` extension. | API returns deterministic error payload: `{ "error_code":"INVALID_PDF", "message":"<human-readable>", "recoverable": false }`; no graph persisted. | **Pass** if error code and shape exactly match and persistence check confirms zero writes. **Fail** otherwise. | 1) Submit invalid fixture. 2) Confirm HTTP status is in configured client-error class. 3) Query datastore for created graph IDs (expect none). |
| Error path | `error_invalid_llm_json` | Inject mock LLM response with malformed JSON for extraction step. | API returns `{ "error_code":"INVALID_LLM_JSON", "message":"<human-readable>", "recoverable": true, "retryable": true }`; pipeline aborts safely with structured logs. | **Pass** if response shape and retry flags match and no partial graph commit occurs. **Fail** otherwise. | 1) Enable invalid-JSON mock. 2) Run extraction. 3) Verify response + logs include parser failure metadata. |
| Error path | `error_unknown_component_query` | Query: “Show dependencies for component `FluxCapacitor`.” | API returns `{ "error_code":"UNKNOWN_COMPONENT", "message":"Component not found", "suggestions":["<known component>"] }`. | **Pass** if `UNKNOWN_COMPONENT` is returned with non-empty suggestions list. **Fail** if null/empty suggestions or wrong code. | 1) Run query with nonexistent component. 2) Verify error contract. 3) Manually check suggested component exists in graph index. |
| Performance (ingestion) | `perf_parse_per_page` | Benchmark parse throughput on 10–50 page fixtures over 5 runs. | Parse latency target: `p95 <= 2.0s/page`; extraction success rate `>= 99%`. | **Pass** if both p95 and success targets are met for two consecutive benchmark runs. **Fail** otherwise. | 1) Execute perf harness with warm cache disabled. 2) Record p50/p95/p99. 3) Attach run artifacts and environment metadata. |
| Performance (query) | `perf_query_latency` | Benchmark top 20 representative queries on populated graph. | Query latency target: `p95 <= 1200ms`, `p99 <= 2000ms`; answer contract validity `100%`. | **Pass** if latency and contract checks pass in the same run. **Fail** on any miss. | 1) Run query benchmark suite. 2) Validate each response against schema. 3) Store report and manually review the slowest 3 queries. |
| Release gate | `mvp_acceptance_gate` | Aggregate all tests above in CI + manual spot check. | All mandatory tests green; no Sev-1/Sev-2 defects open; manual protocol checklist complete with reviewer sign-off. | **Pass** if 100% mandatory automated tests pass and manual checklist is signed by QA + PM. **Fail** if any mandatory check is red or unsigned. | 1) Review CI report. 2) Execute manual checklist items for one happy path, one error path, and one performance report. 3) Capture approver names/date in release log. |

## Cross-page resolution

To improve wiring-reference linking across multi-page schematics, add a dedicated cross-page resolution flow:

1. **Parse reference tokens**
   - Extract explicit page pointers (for example, `to page 3`) and connector/component reference IDs from OCR text.
   - Normalize parsed tokens (page numbers, connector IDs, and directional phrases) into a canonical reference object.

2. **Perform candidate matching heuristics**
   - Match candidate targets using a weighted strategy:
     - connector/component label equality,
     - wire color equality or compatible aliases,
     - spatial proximity to known anchor regions on the referenced page.
   - Keep top candidates with confidence scores for later reconciliation.

3. **Persist unresolved links separately**
   - If no match exceeds confidence threshold, store the reference in an `unresolved_links` collection.
   - Include source page, source element, parsed tokens, attempted candidates, and failure reason.

4. **Run retry strategy with second-pass global matching**
   - After first-pass page-local matching completes, execute a second-pass global resolver over all pages.
   - Retry unresolved links using full-document context (newly discovered connectors, merged aliases, and global graph topology).
   - Promote resolved links back into the canonical link graph; retain unresolved items otherwise.

5. **Expose unresolved count in API outputs**
   - `/upload` response should include `unresolved_reference_count` (and optionally unresolved IDs for debugging).
   - `/metrics` should publish unresolved reference totals/gauges so operators can track reconciliation quality over time.
