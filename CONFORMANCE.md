# AKS Conformance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

All v0.1.0 conformant implementations remain conformant with v0.1.0. All new requirements in v0.1.0 are OPTIONAL unless explicitly marked otherwise.

---

## Required (MUST)

### Exported Stack Format

- [ ] Every exported bundle MUST include `aks_version: "0.1.0"` at the top level
- [ ] Every exported bundle MUST include `scope` set to one of `stack`, `workspace`, or `domain`
- [ ] `entity_count` MUST equal the length of the `entities` array
- [ ] `relationship_count` MUST equal the length of the `relationships` array
- [ ] Every entity MUST include `id` (UUID), `name`, `type`, and `confidence`
- [ ] Every entity MUST include a `source` block
- [ ] Every relationship MUST include `id`, `from_entity_id`, `to_entity_id`, `type`, and `confidence`
- [ ] Every relationship MUST include a `source` block
- [ ] `from_entity_id` and `to_entity_id` MUST reference UUIDs present in the `entities` array
- [ ] Relationships MUST reference entity IDs, not entity names
- [ ] `source.verified` MUST default to `false` for machine-extracted knowledge
- [ ] `source.verified` MUST be `true` on knowledge units created via approved backfeed
- [ ] When filter parameters are applied to export, the bundle MUST include a `filters_applied` field
- [ ] Filtered bundles MUST remain internally consistent: relationships referencing excluded entities MUST also be excluded

### Endpoints

- [ ] `GET /aks/v1/entities` MUST return a list of `AKSEntity` objects
- [ ] `GET /aks/v1/entities` MUST accept `stack_id` or `domain` as query parameters
- [ ] `GET /aks/v1/traverse` MUST perform BFS traversal from a named entity
- [ ] `GET /aks/v1/traverse` MUST accept `start`, `stack_id` or `domain`, and `max_depth` (1-5)
- [ ] `POST /aks/v1/context` MUST accept a context request and return a context result
- [ ] Context result MUST include `traversal_path`, `entities`, `relationships`, `reasoning`, `confidence`, and `source_type`
- [ ] `traversal_path` MUST be an ordered list of entity names the system traversed
- [ ] `GET /aks/v1/export` MUST return a valid `AKSBundle`
- [ ] `GET /aks/v1/export` MUST set a `Content-Disposition: attachment` header
- [ ] `GET /aks/v1/pages/{entity_name}` MUST return an `AKSEntityPage`
- [ ] `GET /aks/v1/pages` MUST return all entity pages as a downloadable zip
- [ ] `POST /aks/v1/backfeed/flag` MUST accept an `AKSBackfeed` and queue it for review
- [ ] `POST /aks/v1/backfeed/approve` MUST absorb approved items with `source.verified = true` and `source.source_kind = "backfeed"`

### Error Handling

- [ ] Every endpoint MUST return HTTP 422 when neither `stack_id` nor `domain` is provided
- [ ] The 422 response MUST include a human-readable message explaining what is required
- [ ] `GET /aks/v1/traverse` MUST return HTTP 404 when the named start entity does not exist
- [ ] `GET /aks/v1/pages/{entity_name}` MUST return HTTP 404 when the named entity does not exist

---

## Recommended (SHOULD)

### Compiled Layer

- [ ] Document quality audits SHOULD run before ingestion and results SHOULD be stored in `AKSDocument.audit`
- [ ] Every agent run SHOULD be logged as an `AKSRun` with all required fields
- [ ] Exported bundles SHOULD include a `runs` array
- [ ] `document_count` SHOULD be included in exported bundles
- [ ] `source.source_kind` SHOULD be populated on every entity and relationship source block
- [ ] For entities extracted from documents: `source.document_id` SHOULD be set
- [ ] For entities extracted from connectors: `source.connector_id` SHOULD be set
- [ ] Entity descriptions SHOULD be synthesized accounts reflecting the entity's role in the full domain, not raw per-document extractions
- [ ] `contributing_documents` SHOULD be populated on entities to enable sourced entity pages
- [ ] `last_seen_at` SHOULD be updated whenever a new document consolidates into an entity
- [ ] `source_hash` SHOULD be populated on `AKSDocument` to enable change detection
- [ ] `truncated` and `original_length` SHOULD be populated when source content exceeds extraction limits
- [ ] Implementations that track compilation strategy SHOULD populate `compilation_strategy` on exported bundles

### Retrieval Quality

- [ ] Implementations SHOULD combine vector similarity and text similarity using geometric mean scoring
- [ ] Implementations SHOULD apply a neutral text score when trigram similarity is below threshold for short queries
- [ ] Implementations SHOULD apply a recency boost multiplier that gives mild preference to chunks from recently ingested documents

### Entity Pages

- [ ] Entity pages SHOULD be generated on demand from the current graph state, not cached without invalidation
- [ ] Entity pages SHOULD include `graph_version` to enable cache invalidation
- [ ] Entity pages SHOULD include `source_type`
- [ ] Hybrid entity pages SHOULD clearly label live content with connector name and `live_at` timestamp
- [ ] Entity pages SHOULD include a Sources section when `contributing_documents` is populated
- [ ] `GET /aks/v1/backfeed/queue` SHOULD be available and return pending items

### Connectors

- [ ] Stacks with connectors SHOULD include `connector_count` in exported bundles
- [ ] Stacks with connectors SHOULD include a `connectors` array in exported bundles
- [ ] Sync-mode connector data SHOULD go through the full ingestion pipeline
- [ ] Live-mode connector data SHOULD NOT be absorbed into the compiled ontology without explicit human approval

---

## Optional (MAY)

- [ ] Implementations MAY cache entity pages if cache is invalidated when `graph_version` changes
- [ ] Implementations MAY include `recommended_tools` on exported bundles
- [ ] Implementations MAY expose AKS endpoints as MCP tools
- [ ] `POST /aks/v1/context` MAY accept `include_live: true` to incorporate live connector data
- [ ] Hybrid entity pages MAY be generated when `include_live: true` is requested
- [ ] `GET /aks/v1/traverse` MAY accept a `relationship_types` filter parameter
- [ ] Implementations MAY support multi-stack composition via Import, Reference, or Federated Query patterns
- [ ] Implementations MAY offer both incremental and global compilation strategies

---

## Verifying Conformance

### Validate a bundle against the JSON Schema

    pip install jsonschema
    python3 -c "
    import json, jsonschema
    schema = json.load(open('SCHEMA.json'))
    stack = json.load(open('exported-stack.json'))
    jsonschema.validate(stack, schema)
    print('Conformant')
    "

### Check for required endpoints

    for path in \
      "/aks/v1/entities?domain=test" \
      "/aks/v1/export?domain=test" \
      "/aks/v1/backfeed/queue?domain=test"; do
      status=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:8000$path")
      echo "$path -> $status"
    done

### Check 422 on missing parameters

    curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/aks/v1/entities
    # Expected: 422

### Check source_type in context response

    curl -s -X POST http://localhost:8000/aks/v1/context \
      -H "Content-Type: application/json" \
      -d '{"query":"test","domain":"your-domain"}' \
      | python3 -c "import json,sys; r=json.loads(sys.stdin.read()); print('source_type:', r.get('source_type'))"
    # Expected: source_type: compiled

### Check traversal returns 404 for unknown entity

    curl -s -o /dev/null -w "%{http_code}" \
      "http://localhost:8000/aks/v1/traverse?start=NonExistentEntity&domain=your-domain"
    # Expected: 404

---

## Versioning

A change is a breaking change if it removes a required field, changes the type of an existing field, changes the meaning of an existing enum value, removes a required endpoint, or changes a required endpoint's response shape. Breaking changes require a major version bump.

Non-breaking additions (new OPTIONAL fields, new OPTIONAL endpoints) MAY be introduced in minor versions. All v0.1.0 implementations remain conformant with v0.1.0.
