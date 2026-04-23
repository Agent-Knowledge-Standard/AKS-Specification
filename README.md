# AKS, the Agent Knowledge Standard

**version 0.1.0**

The **Agent Knowledge Standard** is the open standard for *compiled domain knowledge*. It defines the format and query interface of a Knowledge Stack: a smart, structured, persistent artifact that any agent or tool can read without rebuilding domain understanding from scratch.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## Contents

1. [The Problem](#the-problem)
2. [What Is A Knowledge Stack](#what-is-a-knowledge-stack)
3. [Knowledge Stack and AKS Bundle](#knowledge-stack-and-aks-bundle)
4. [The Two Layers](#the-two-layers)
5. [Relationship to Agent Memory](#relationship-to-agent-memory)
6. [Architecture](#architecture)
7. [The Knowledge Unit](#the-knowledge-unit)
8. [Connected Sources](#connected-sources)
9. [The Export Format](#the-export-format)
10. [The Query Interface](#the-query-interface)
11. [Entity Pages](#entity-pages)
12. [Retrieval Quality](#retrieval-quality)
13. [The Run Record](#the-run-record)
14. [Backfeed](#backfeed)
15. [Document Audit](#document-audit)
16. [Scope and Graduation](#scope-and-graduation)
17. [Stack Composition](#stack-composition)
18. [Consumption Paths](#consumption-paths)
19. [MCP Integration](#mcp-integration)
20. [Conformance](#conformance)
21. [Prior Art](#prior-art)
22. [Versioning](#versioning)
23. [Contributing](#contributing)
24. [License](#license)


---

## The Problem

The teams getting the most out of AI right now are the ones whose agents actually understand their domain. That's the advantage worth having, and every team has to build it from scratch. Getting an agent to understand your business takes real work: defining concepts, mapping relationships, tuning retrieval, wiring it all to a model. What you end up with is useful, and it's also completely stuck inside the pipeline you built it in.

There are two concerns with this: The first is that nothing carries forward. Switching tools resets the work, new teams have to remake it, and every new session starts the agent back at zero. The second problem is that the work itself is built on the wrong foundation. Most systems treat knowledge as text chunks that are matched by vector similarity at query time, but while vectors might cache text representations, they do not cache relational understanding of the world around us. 

A capable AI agent can reason through that a **Request** *requires* a **Policy** which *governs* a **Required Approval** when it has the right context in front of it, but that chain only exists in the model's working memory for that session. Nothing about the retrieval layer preserves it. Every session reassembles the same relationships from scratch, and every tool that needs those relationships has to rebuild them independently.

The Agent Knowledge Standard defines the format for taking that work out of the pipeline and putting it somewhere any agent can read it. Domain knowledge is compiled into a structured graph of entities, typed relationships, confidence scores, and source traceability. Any AKS-aware tool can query that graph, so the work survives the pipeline that produced it and the relational understanding survives the session that needed it.

---

## What Is A Knowledge Stack

A **Knowledge Stack** is not a knowledge base. The distinction is fundamental and intentional.

A knowledge base stores documents. You search it and it returns text that looks related to your query. It does not understand what it contains; it retrieves what matches. It does not know what questions have been asked of it, what it does not cover, or how its contents relate to each other. Confluence is a knowledge base. Notion is a knowledge base. A folder of PDFs with a search box is a knowledge base.

A Knowledge Stack compiles source material into structured, persistent understanding. It knows what entities exist in a domain, how they relate to each other, which relationships are well-supported and which are thin, and which questions agents have already asked and at what confidence. It gets smarter from usage. It knows its own gaps. It exposes a standard query interface so any agent or tool can read from it without rebuilding domain understanding from scratch.

The word "stack" is deliberate. It describes layers of compiled knowledge, not a pile of stored content:

```
Source Material   - documents, connected sources (Slack, Jira, etc.)
      |
      | compiles into
      |
      v  
Domain Ontology   - entities, typed relationships, confidence, source traceability
      |
      | exposes
      |
      v  
Query Interface   - AKS endpoints that agents and tools should call
      |
      | derives
      |
      v  
Entity Pages   - human-readable pages generated from the graph on demand
```

### Two People, One Stack

A Knowledge Stack is most valuable when it is built by more than one kind of contributor. The same Stack serves different roles and gets stronger from each of them.

Consider a support lead and a developer working at the same company. Both need to understand the company's Billing system, and both contribute to the Billing Stack in different ways.

The **support lead** lives in the domain every day. She knows which customer tiers get which discounts, which edge cases trigger manual invoicing, which exceptions exist for enterprise contracts, and how the team actually handles a refund request when the automated flow fails. Her contribution to the Stack comes from internal runbooks, customer escalation notes, Slack threads from incident retrospectives, and the occasional direct edit to correct an extracted entity. She sees Billing as a set of policies, exceptions, and customer outcomes.

The **developer** works on a sliver of the Billing service. He knows how the refund API is structured, which database tables back the invoicing flow, what events get emitted when a subscription upgrades, and which edge cases his service handles versus passes to the payment processor. His contribution comes from the codebase's ADRs, API specs, deployment runbooks, and PR descriptions. He sees Billing as a set of services, data models, and integration points.

Neither of them has a complete picture on their own. The support lead cannot reason about how a bug in the refund API surfaces downstream. The developer cannot reason about why a particular discount policy exists or which customers it applies to. An agent querying either person's mental model would get half the answer.

When both feed the same Stack, the compiled ontology knows that the "Refund" entity is connected to the "Refund API", "Refund Policy", "Customer Tier Discounts", and "Manual Invoicing Exception" entities. An agent answering a question about refunds for an enterprise customer can traverse across the policy, the code, and the exceptional cases without having to know which team owns which piece. The Stack becomes a stronger resource than either contributor could have produced alone, and stronger than the sum of what each of them knows in isolation.

This is the compounding effect of a Knowledge Stack: different roles contribute different facets of the same domain, and the compiled graph holds them together.

A knowledge base waits to be searched. A Knowledge Stack accumulates understanding, compounds from usage, and makes what a domain knows available to any tool through a standard interface, without each tool rebuilding that understanding independently.

AKS defines the compiled ontology and the query interface. How knowledge is compiled from source material is implementation-defined. What agents do with the knowledge they receive is also implementation-defined. AKS governs the layer in between.

Put simply: **compiled domain knowledge is a Knowledge Stack.**

---

## Knowledge Stack and AKS Bundle

**A Knowledge Stack is a live system.** It has a `stack_id`, a `domain`, a `scope`, a compilation history, and a set of AKS-compliant endpoints it exposes. It is not a file; it is a running artifact that gets smarter over time.

**An AKS Bundle is a portable snapshot of a Knowledge Stack.** It is what `GET /aks/v1/export` produces: a single JSON file containing the compiled entities, relationships, source records, and run history at a point in time. It can be validated against the JSON Schema, imported into another system, or archived. It is not the Stack; it is what the Stack produces when asked to export itself.

In this spec: requirements that say "the Stack MUST..." apply to the live system. Requirements that say "the bundle MUST include..." apply to the exported JSON file.

---

## The Two Knowledge Layers

A Knowledge Stack produces two outputs from one compilation pass.

**Machine-readable graph.** A queryable structure of entities with typed relationships, confidence scores, and source traceability. This is what agents traverse at query time. When an agent answers a question, it follows typed relationships through the graph rather than pattern-matching text. The path it follows is explicit, auditable, and repeatable.

**Human-readable entity pages.** One page per entity, generated from the graph on demand. Pages MUST be derived from the graph and MUST always reflect its current state. They MUST NOT be written manually or stored independently. In v0.1.0, pages MAY also incorporate live connector data, producing hybrid pages.

The graph is what agents read. The entity pages are what humans read. Both come from the same compiled source.

---

## Relationship to Agent Memory

Agent memory and Knowledge Stacks are related but distinct.

**Memory is first-person and temporal.** It belongs to one agent, tracks what that agent experienced, and is organized around when things happened. Memory answers: what did this agent encounter? It is session-scoped and agent-specific by nature. When the agent is replaced or reset, the memory goes with it.

**A Knowledge Stack is third-person and structural.** It belongs to a domain, tracks what that domain knows, and is organized around concepts and relationships. A Stack answers: what does this domain know, and how do its concepts relate? It persists and compounds regardless of which agent queried it last.

The two layers compose naturally. Agent memory is raw material. A Knowledge Stack is compiled infrastructure.

```
Agent session
    |
    | agent produces a useful answer
    | or surfaces a new relationship
    |
    v
Backfeed queue  (flagged for human review)
    |
    | approved
    |
    v
Knowledge Stack  (compiled, verified, permanent)
    |
    | provides
    |
    v
Shared context for any agent that queries the Stack
```

Memory gives one agent shared context with its past self. A Knowledge Stack gives many agents shared context with a domain. AKS is not a memory standard. It is the standard for the compiled layer that memory feeds into.

---

## Architecture

```
Source Material   - documents, connected sources (Slack, Jira, etc.)
      |
      | compiles into
      |
      v  
Domain Ontology   - entities, typed relationships, confidence, source traceability
      |
      | exposes
      |
      v  
Query Interface   - AKS endpoints that agents and tools should call
      |
      | derives
      |
      v  
Entity Pages   - human-readable pages generated from the graph on demand
```

---

## The Knowledge Unit

Every entity and relationship MUST carry four things: identity, meaning, confidence, and source.

**Entity**

```json
    {
      "id": "uuid",
      "name": "string",
      "type": "string",
      "description": "string",
      "properties": { "key": "value" },
      "confidence": 0.0 to 1.0,
      "contributing_documents": [ "uuid" ],
      "last_seen_at": "ISO8601",
      "source": {
        "document_id": "uuid",
        "connector_id": "uuid",
        "source_kind": "document | connector | backfeed | manual",
        "extracted_at": "ISO8601",
        "model": "string",
        "verified": false,
        "pipeline": "string"
      }
    }
```

**Relationship**

```json
    {
      "id": "uuid",
      "from_entity_id": "uuid",
      "to_entity_id": "uuid",
      "type": "verb phrase",
      "description": "string",
      "confidence": 0.0 to 1.0,
      "source": { ... }
    }
```

Relationships MUST reference entity IDs, not names. The `source.verified` field defaults to `false` for machine-extracted knowledge. The scope graduation protocol is gated on verified entity thresholds.

### Entity Field Reference

| Field | What it records |
|---|---|
| `id` | Stable UUID. Does not change when the entity is renamed |
| `name` | Human-readable display name |
| `type` | Domain-defined category (Concept, Process, Role, System, etc.). No global enum |
| `description` | Compiled account of the entity within the full domain. See Entity Descriptions and Domain Context below |
| `properties` | Arbitrary key-value metadata specific to this entity |
| `confidence` | Extraction quality, 0.0 to 1.0. Increases as more sources corroborate the entity |
| `contributing_documents` | UUIDs of every source document that has contributed to this entity through compilation or consolidation |
| `last_seen_at` | Timestamp of the most recent document ingestion that mentioned or reinforced this entity. A staleness signal. See Entity Staleness Signal below |
| `source` | Traceability block. See Source Traceability below |

### Relationship Field Reference

| Field | What it records |
|---|---|
| `id` | Stable UUID |
| `from_entity_id` | UUID of the entity this relationship originates from |
| `to_entity_id` | UUID of the entity this relationship points to |
| `type` | Verb phrase describing the relationship (requires, produces, governs, etc.). Domain-defined |
| `description` | Optional prose describing the relationship in context |
| `confidence` | Extraction quality, 0.0 to 1.0 |
| `source` | Traceability block. Same shape as entity source |

### Source Traceability Reference

The `source` block on every entity and relationship records how a knowledge unit was produced.

| Field | What it records |
|---|---|
| `document_id` | UUID of the source document. Present when `source_kind` is `document` |
| `connector_id` | UUID of the connector. Present when `source_kind` is `connector` |
| `source_kind` | How the knowledge unit was produced: `document`, `connector`, `backfeed`, or `manual` |
| `extracted_at` | Timestamp when the knowledge unit was extracted |
| `model` | Identifier of the model that performed extraction (e.g. `claude-sonnet-4-6`) |
| `verified` | Whether a human has reviewed and confirmed this knowledge unit. Defaults to `false` for machine-extracted knowledge. The unit of graduation evidence |
| `pipeline` | Identifier of the extraction pipeline (e.g. `llamaindex-schema-extraction`, `backfeed`, `manual`) |

Exactly one of `document_id` or `connector_id` SHOULD be present when `source_kind` is `document` or `connector` respectively.

### Entity Descriptions and Domain Context

The `description` field is not a summary of what one document says about an entity. It is the entity's compiled account within the context of the full domain.

A well-compiled entity description reflects two things simultaneously: what the source material says about this entity, and how this entity relates to everything else the Stack knows. A description of "Mitochondria" that reads "an organelle" is thin. One that reads "the organelle responsible for ATP synthesis, functionally dependent on Glucose and Oxygen, and producing byproducts absorbed by the Cytoplasm" is rich, because the compiler knew those sibling entities existed when it wrote the description.

Implementations SHOULD establish the full domain vocabulary across all source material before generating individual entity descriptions. A compilation pass that processes one document in isolation produces weaker descriptions than one that first surveys all concepts across all sources and then generates each description knowing what its neighbors are.

The `description` field SHOULD NOT be the raw extraction output of a single LLM call against a single chunk. It SHOULD be a synthesized account that reflects the entity's role within the full compiled domain.

### Compilation Strategy

Implementations MAY offer two compilation strategies:

**Incremental.** Each document is processed as it arrives. Fast feedback; entities improve through consolidation but lack full domain vocabulary awareness at first extraction.

**Global.** All documents are processed as a batch through a two-phase pipeline. Phase one establishes the domain vocabulary. Phase two generates each entity description knowing every sibling concept. Triggered manually as a recompile action. Produces richer descriptions.

The two are not mutually exclusive. A reasonable pattern is incremental compilation on upload for fast feedback, with a manual recompile that runs the global pass when the Stack has reached critical mass.

### Entity Staleness Signal

The `last_seen_at` field on `AKSEntity` records the most recent document ingestion that mentioned or consolidated into this entity. Consumers MAY use this as a staleness signal.

Staleness is surfaced as information, not applied automatically. The Stack owner decides whether an entity that has not appeared in new sources for 90 days warrants review, re-ingestion, or no action. The field provides the signal; the policy is the owner's to set.

This is distinct from `confidence`, which measures extraction quality. An entity can have high confidence and be stale, or low confidence and be current. They are independent signals.

---

## Connected Sources

Knowledge Stacks support connected sources alongside uploaded documents. Connected sources are live data systems: Slack, Jira, GitHub, Salesforce, Notion, or any database with a connector implementation.

Connectors operate in one of two modes:

**Sync mode.** Data is periodically pulled and ingested into the compiled layer through the same pipeline as documents. The ontology grows richer over time as connector data is compiled.

**Live mode.** Data is fetched at agent runtime and injected directly into context without a compilation pass. Live data is not in the vector store and has not been through verification or consolidation.

The `source_type` field on `AKSRun` and `AKSEntityPage` distinguishes compiled, live, and hybrid outputs. A HIGH confidence score on a hybrid run carries a different trust level than the same score on a compiled run.

---

## The Export Format

A full Knowledge Stack MUST be exportable as an AKS Bundle.

```json
    {
      "aks_version": "0.2.0",
      "domain": "lowercase-hyphenated-slug",
      "stack_id": "uuid",
      "stack_name": "string",
      "compiled_at": "ISO8601",
      "scope": "stack | workspace | domain",
      "entity_count": integer,
      "relationship_count": integer,
      "document_count": integer,
      "connector_count": integer,
      "compilation_strategy": "incremental | global",
      "documents": [ AKSDocument ],
      "connectors": [ AKSConnector ],
      "entities": [ AKSEntity ],
      "relationships": [ AKSRelationship ],
      "runs": [ AKSRun ],
      "filters_applied": { ... }
    }
```

`entity_count` MUST equal the length of `entities`. `relationship_count` MUST equal the length of `relationships`.

### Bundle Size and Stack Scale

**Bundle scale.** The Bundle is a JSON snapshot delivered as a file, with size limits at two layers.

First, bundles have a raw size footprint. A typical entity serializes to roughly 1KB including its description, source block, and relationships metadata. A relationship is around 500 bytes. A Stack with 5,000 entities and 8,000 relationships produces a bundle in the 10MB range. At 20,000 entities and 40,000 relationships the bundle approaches 50MB. At that scale the file is slow to transmit, slow to parse, and expensive to hold in memory regardless of how it is consumed. [AI made did this reasoning here, I need to verify these numbers, but intuitively this makes sense.]

Second, bundles have consumer-imposed limits beyond raw size:

| Path | Practical ceiling | Reason |
|------|-----------------|--------|
| Path 1: MCP Server | No hard ceiling | Server queries the live Stack |
| Path 2: Managed MCP | No hard ceiling | Live Stack queries |
| Path 3: Project Knowledge Base | ~500 entities | Context window constraint |

Even the paths marked "no hard ceiling" still pay a cost on full bundle exports. A Stack too large for its primary consumption path should be exported with filter parameters (`min_confidence`, `verified_only`, `entity_type`, `max_entities`) to produce a focused subset. Full unfiltered exports make sense for backup and migration; filtered exports make sense for day-to-day consumption.

---

## The Query Interface

AKS-compliant implementations MUST expose the following endpoints. Every endpoint MUST require either `stack_id` (UUID) or `domain` (slug). If neither is provided the endpoint MUST return HTTP 422.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /aks/v1/entities | List compiled entities. MAY filter by type and min_confidence. |
| GET | /aks/v1/traverse | BFS traversal. Params: start, max_depth (1-5), relationship_types. |
| POST | /aks/v1/context | Assemble the most relevant subgraph for a natural language query. |
| GET | /aks/v1/export | Export as AKS Bundle. Filter params: min_confidence, verified_only, entity_type, max_entities. |
| GET | /aks/v1/pages/{entity_name} | Return the human-readable entity page for a named entity. |
| GET | /aks/v1/pages | Export all entity pages as a zip. |
| POST | /aks/v1/backfeed/flag | Flag content for backfeed. |
| POST | /aks/v1/backfeed/approve | Absorb an approved backfeed item into the ontology. |
| GET | /aks/v1/backfeed/queue | List pending backfeed items. |

### POST /aks/v1/context

Request:

```json
    {
      "query": "natural language question",
      "stack_id": "uuid",
      "domain": "slug",
      "max_entities": 8,
      "include_relationships": true,
      "include_live": false
    }
```

Response:

```json
    {
      "query": "string",
      "entities": [ AKSEntity ],
      "relationships": [ AKSRelationship ],
      "traversal_path": [ "Entity1", "Entity2", "Entity3" ],
      "reasoning": "string",
      "confidence": "HIGH | MEDIUM | LOW",
      "source_type": "compiled | live | hybrid"
    }
```

The `traversal_path` MUST contain the ordered entity names the system traversed. The `source_type` field MUST be present in responses.

### GET /aks/v1/traverse

The `start` parameter MUST be an entity name. The `max_depth` parameter MUST be an integer between 1 and 5 inclusive. Returns HTTP 404 when the named entity does not exist. The optional `relationship_types` parameter filters which relationship types to follow, enabling focused traversal without LLM cost.

### GET /aks/v1/export

Supports optional filter parameters: `min_confidence`, `verified_only`, `entity_type`, `max_entities`. When filters are applied, the bundle MUST include a `filters_applied` field. Relationships whose `from_entity_id` or `to_entity_id` references an excluded entity MUST also be excluded.

---

## Entity Pages

Entity pages MUST be generated from the compiled graph on demand. They MUST NOT be stored independently. They MUST always reflect the current graph state.

An `AKSEntityPage` MUST contain:

```json
    {
      "entity_name": "string",
      "markdown": "full markdown content",
      "generated_at": "ISO8601",
      "source_entity_id": "uuid",
      "graph_version": "string",
      "source_type": "compiled | live | hybrid",
      "live_sources": [ "connector_uuid" ],
      "live_at": "ISO8601"
    }
```

### How Entity Pages Are Generated

Entity pages are built from compiled ontology data, not from the original source documents. The implementation assembles the entity's compiled attributes (name, type, description, properties, confidence, relationships, and contributing document metadata) and passes this structured data to a language model to synthesize a human-readable markdown page.

Because entity descriptions SHOULD be compiled with full domain context, a well-generated entity page reads like a wiki page written by someone who understands the whole domain. The page for "Incident" in a software engineering Stack naturally references Monitoring, On-Call, Rollback, and Postmortem because those entities exist in the compiled graph and the description was written knowing they were there.

### Source Citations on Entity Pages

When `contributing_documents` is populated on an entity, implementations SHOULD include a Sources section on the generated page. Each contributing document is listed with its filename and ingestion date. Where possible, the generated prose notes which source established a key claim using inline citations.

### Reconstruction From A Bundle

Because entity pages derive entirely from the compiled ontology, they can be generated from an exported AKS Bundle without access to the original source documents. The Bundle contains all entities, all relationships, all confidence scores, and all source traceability. An AKS Bundle is self-contained for the purpose of knowledge consumption.

If the original source documents are also available, the compilation pipeline can be re-run with a different or newer inference model for potentially richer output.

---

## Retrieval Quality

The quality of `POST /aks/v1/context` depends on how well implementations rank document chunks before entity identification. Two retrieval improvements are RECOMMENDED:

**Geometric mean scoring.** Rather than ranking chunks by vector similarity alone, combine vector similarity and text (trigram) similarity using a geometric mean. A chunk only ranks highly when both signals agree. This prevents semantically adjacent but textually irrelevant chunks from surfacing, and prevents textually matching but semantically distant chunks from being suppressed.

    score = sqrt(vector_similarity * text_similarity)

When the query is very short (fewer than three words), trigram similarity scores poorly due to insufficient n-gram overlap. Implementations SHOULD apply a neutral text score (0.5) when trigram similarity falls below a threshold, rather than pulling the geometric mean toward zero.

**Recency boost.** When chunks have similar similarity scores, prefer those from more recently ingested documents. A soft decay from 1.0 toward a floor value (e.g. 0.85) over a configurable window (e.g. 365 days) gives fresh content a mild edge without suppressing older knowledge. Old chunks still surface, they just lose a small advantage against recent ones.

    final_score = geometric_mean_score * recency_multiplier

These are RECOMMENDED implementation patterns, not REQUIRED. The spec governs the endpoint response shape, not the internal ranking algorithm. Both patterns are described in DECISIONS.md with their tradeoffs.

---

## The Run Record

Every agent run against a Stack SHOULD produce an `AKSRun` record.

```json
    {
      "run_id": "uuid",
      "stack_id": "uuid",
      "aks_version": "0.2.0",
      "trigger": "manual | auto | api",
      "query": "string",
      "traversal_path": [ "string" ],
      "entities_used": [ "string" ],
      "confidence": "HIGH | MEDIUM | LOW",
      "source_type": "compiled | live | hybrid",
      "origin": "run | direct",
      "timestamp": "ISO8601"
    }
```

Low confidence runs on specific entities signal where the Stack needs more source material. Implementations SHOULD surface this to Stack owners. The `source_type` field helps interpret confidence: HIGH on a compiled run reflects verified ontology quality; HIGH on a hybrid run also includes live unverified data.

---

## Backfeed

Backfeed is the process of feeding agent run outputs or flagged content back into the compiled ontology. A human MUST review it before it is absorbed. On approval, entities and relationships are added with `source.verified = true` and `source.source_kind = "backfeed"`.

```json
    {
      "content": "string",
      "origin": "run | direct",
      "stack_id": "uuid",
      "flagged_by": "string",
      "source_run_id": "uuid"
    }
```

The name *backfeed* is borrowed from electrical systems. Backfeed is power flowing the opposite direction its supposed to, like a home with solar panels pushing excess electricity back into the grid. Without the right safety gate it's dangerous, and with one it's valuable. AKS backfeed works the same way. Without the human review gate, agent output flowing back into the ontology corrupts it. With the gate, the same output compounds the Stack's value. The word carries that tradeoff on purpose.

---

## Document Audit

Before a document is ingested, implementations SHOULD run a quality audit and store the results in `AKSDocument.audit`. The audit gives Stack owners a signal about whether a document is worth ingesting before it enters the ontology. Silently accepting every upload leads to Stacks cluttered with near-duplicates, outdated information, and contradictions that quietly degrade retrieval quality over time. This has the added benefit that it can also aid in RAG implementations that pull from the source documents directly rather than the compiled graph, by flagging low-quality documents before they contribute to noisy retrieval.

An `AKSDocumentAudit` records:

- **`duplicate_score`** (0.0 to 1.0): similarity to content already in the Stack. Near-duplicate uploads inflate entity counts without adding information.
- **`staleness_score`** (0.0 to 1.0): estimated currency of the document. A handbook from three years ago may still be valid, or may be entirely outdated.
- **`quality_classification`**: one of `healthy`, `redundant`, `outdated`, `trivial`, or combinations such as `redundant+outdated`. Drives the `recommendation` field.
- **`contradiction_count`** and **`contradiction_flags`**: specific conflicts with existing entities, with severity.
- **`recommendation`**: `proceed`, `review`, or `dismiss`.

Implementations SHOULD surface the recommendation to a Stack owner before ingestion rather than silently proceeding. A document flagged `dismiss` for being a near-duplicate of existing content is better filtered out than compiled into yet another consolidation pass.

### Document Hygiene

Three optional fields on `AKSDocument` track what was ingested, not how good it was. They are independent of the quality audit but serve essential roles in Stack maintenance.

**`source_hash`.** SHA-256 hash of the source content at ingestion. This is how implementations detect whether a re-uploaded document has actually changed. A user who uploads the same handbook twice, or a connector that re-syncs the same Notion page, produces identical content hashes, and implementations MAY skip extraction entirely rather than running the full compilation pipeline on unchanged material. Without this hash there is no reliable way to tell a genuinely updated document from a redundant re-upload.

**`truncated`.** Boolean indicating whether the source was truncated before extraction due to context window limits. When true, the document was not fully processed, and entities extracted from it may be missing information that existed in the source but never reached the compilation step. Consumers SHOULD interpret entities from truncated sources with reduced confidence.

**`original_length`.** Character count before truncation. Present only when `truncated` is true. Surfaces how much of the source was cut so Stack owners can decide whether to chunk the document differently and re-ingest.

---

## Scope and Graduation

| Scope | Visibility | Graduation requirement |
|-------|------------|------------------------|
| `stack` | Private to this Stack | Default for all new Stacks |
| `workspace` | Shared within an organization | Human review of extracted entities |
| `domain` | Public, community-governed | Threshold of verified entities and community approval |

The `source.verified` field is the unit of graduation evidence.

---

## Stack Composition

A single agent or tool MAY consume multiple Knowledge Stacks. Three composition patterns are supported:

**Import.** Stack B ingests Stack A's exported bundle through the standard compilation pipeline. A's entities are absorbed into B with consolidation. A is unchanged. This is the simplest model.

**Reference.** An agent binds to one primary Stack but consults other Stacks as read-only context sources at query time. Run records and backfeed accumulate on the primary Stack only.

**Federated query.** The consumer queries multiple Stacks at runtime and merges the resulting subgraphs. No Stack knows about the others.

None of these patterns require new spec surface beyond the standard `POST /aks/v1/context` interface. Source traceability (`source.document_id` and `source.connector_id` on every entity and relationship) is what makes composition tractable, including comparison use cases where an agent needs to distinguish which Stack or document a claim originated from.

### Compiling From An Existing Wiki

A Karpathy-style LLM wiki (markdown files with named entity pages and cross-references) can be compiled into a Knowledge Stack. The mapping is direct:

| Wiki concept | AKS equivalent |
|---|---|
| Page title | `entity.name` |
| Page content | `entity.description` |
| Page category | `entity.type` |
| Cross-reference `[[link]]` | `relationship` (type inferred or assigned) |
| The wiki page itself | `source.document_id` reference |

The compilation pass needs to assign UUIDs (stable across renames), type the relationships (from untyped wikilinks to verb phrases), and set confidence and source metadata.

---

## Consumption Paths

A Knowledge Stack can be consumed in three ways. All three draw from the same compiled knowledge:

### Path 1, Stack as MCP Server

```json
    {
      "mcpServers": {
        "aks": {
          "url": "http://localhost:8000/mcp",
          "tools": ["query_stack", "traverse_stack", "get_entities"]
        }
      }
    }
```

### Path 2, Managed MCP Server

```
    mcp.your-provider.com/workspace/{workspace_id}/mcp
```

All Stacks in the workspace are automatically available as tools. No self-hosting required.

### Path 3, AKS Bundle in a Project Knowledge Base

Many AI tools, including Claude Projects, allow users to upload reference material into a persistent project knowledge base that every chat in that project can draw from. An exported Stack works well in this context:

- Export all entity pages with `GET /aks/v1/pages` and upload the resulting markdown files directly to the project's knowledge base
- Export the bundle with `GET /aks/v1/export` and upload the JSON file for tools that accept arbitrary text files

Entity pages are usually the better fit: they are already formatted for human consumption, each one is a focused chunk of compiled knowledge on a single entity, and they carry source citations inline. This path requires no running server and no MCP configuration, but the uploaded material is a static snapshot rather than a live query against the Stack.

As AI tools adopt native support for the AKS Bundle format, this path becomes more direct. A tool that recognizes an AKS Bundle on upload can parse the compiled ontology, surface entity pages natively, and respect source traceability and confidence scores in its interface. Broader adoption of the standard means Stacks become easier to access and maintain across the tools a team already uses, rather than requiring teams to convert compiled knowledge into each tool's preferred format.

---

## MCP Integration

An AKS-compliant MCP server SHOULD expose the following tools at a minimum:

**query_stack** wraps `POST /aks/v1/context`
```
    Input:  { query: string, domain: string, include_live?: boolean }
    Output: { entities[], relationships[], traversal_path[], reasoning, confidence, source_type }
```

**traverse_stack** wraps `GET /aks/v1/traverse`
```
    Input:  { start: string, domain: string, max_depth?: number, relationship_types?: string[] }
    Output: { steps[], entity_count }
```

**get_entities** wraps `GET /aks/v1/entities`
```
    Input:  { domain: string, type?: string, min_confidence?: number }
    Output: { entities[] }
```

---

## Conformance

See [CONFORMANCE.md](./CONFORMANCE.md).

Validate any exported Stack:
```python
    pip install jsonschema
    python3 -c "
    import json, jsonschema
    schema = json.load(open('SCHEMA.json'))
    bundle = json.load(open('exported-stack.json'))
    jsonschema.validate(bundle, schema)
    print('AKS Conformant')
    "
```

---

## Prior Art

This spec is a synthesis of patterns observed across many implementations. The following are the most direct influences, but the ideas in this space are evolving rapidly and no single implementation fully embodies the standard as it stands today. What you see in here is the result of distilling common principles and best practices across the ecosystem and from my own experience working on AI tooling, rather than codifying any one implementation. This list will expand over time, I'm sure of it.

**Karpathy LLM Wiki.** The insight that knowledge should be compiled into a persistent structured artifact. The entity page layer is the formalization of this principle. The backfeed mechanism formalizes his writeback rule. The cross-document consolidation pass is the direct implementation of his maintain-and-update model.

**llm-wiki-compiler.** A TypeScript CLI implementing the Karpathy pattern. Contributes the two-phase vocabulary-first compilation approach and the truncation honesty pattern (`truncated`, `original_length` on AKSDocument).

**BrainDB.** An LLM wiki upgraded to a database, with typed entities, graph relations, and HTTP API. Closest to AKS in architecture. Contributes the geometric mean retrieval scoring pattern and surfaces the value of recency weighting in retrieval. Differs from AKS in scope: BrainDB is agent memory for one agent; AKS is domain knowledge infrastructure for many agents across many tools.

**Mozilla cq.** The trust graduation protocol maps directly to AKS scope graduation from *stack* to *workspace* to *domain*.

**Ontology engineering community.** AKS takes the core insight (agents need structured relational knowledge, not just text similarity) and implements it without requiring OWL, SPARQL, etc.

---

## Versioning

A change is a breaking change if it removes a required field, changes the type of an existing field, changes the meaning of an existing enum value, removes a required endpoint, or changes a required endpoint's response shape. Breaking changes require a major version bump.

All v0.1.0 implementations remain conformant with v0.1.0. All additions in v0.1.0 are OPTIONAL fields.

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). The spec is community-governed. 

---

## License

Apache 2.0. See [LICENSE](./LICENSE).
