# Product Requirements Document (PRD)

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
