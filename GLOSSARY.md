# AKS Glossary

Precise definitions of all terms used in the AKS specification. When a term in this glossary conflicts with everyday usage, the definition here takes precedence.

---

## Agent

A software system that queries a Knowledge Stack at runtime and produces outputs grounded in the compiled knowledge it receives. An agent is a consumer of AKS. It is not defined by AKS and may be implemented in any way.

---

## AKS Bundle

The portable JSON snapshot of a Knowledge Stack, produced by `GET /aks/v1/export`. Represents the compiled knowledge of a Stack at a point in time. Any AKS-aware tool can import a bundle without custom integration. See `SCHEMA.json` for the canonical definition.

---

## AKS-compliant

A system that satisfies all MUST requirements in the AKS conformance checklist. See `CONFORMANCE.md`.

---

## Backfeed

The process of feeding agent run outputs or flagged content back into a compiled Knowledge Stack. Backfeed entries require human review before absorption. On approval, extracted entities and relationships are added with `source.verified = true` and `source.source_kind = "backfeed"`. This is what makes a Stack smarter from usage rather than only from document ingestion.

---

## Compiled knowledge

Domain understanding that has been extracted from raw source material, structured as typed entities and relationships, and stored in persistent queryable form. Compiled knowledge persists across sessions and accumulates over time. It differs from raw document retrieval, where meaning is reassembled from text similarity scores on every query.

---

## Compiled layer

The portion of a Knowledge Stack produced by the compilation pass: entities, relationships, vector embeddings, and source traceability. Always available at query time with no external dependencies. Distinct from the live layer.

---

## Compilation strategy

How a Knowledge Stack was compiled from its source material. Two values are defined:

`incremental` means each document was processed as it arrived, producing entities without full domain vocabulary awareness at first extraction. Descriptions improve through cross-document consolidation but may lack the richness of globally compiled knowledge.

`global` means a two-phase compilation pipeline was run. Phase one surveys the full corpus and establishes the domain vocabulary. Phase two generates each entity description knowing every sibling concept. Produces richer domain-aware descriptions than incremental.

The two are not mutually exclusive. A typical pattern is incremental compilation on upload for fast feedback, with a manual recompile that runs the global pass over all documents.

---

## Confidence

A float between 0.0 and 1.0 expressing how reliably an entity or relationship was extracted. Increases as more source documents corroborate this entity. Distinct from `last_seen_at` (staleness); an entity can have high confidence and be stale, or low confidence and be current.

In run records, confidence is expressed as an enum: `HIGH`, `MEDIUM`, or `LOW`. This reflects the agent's self-reported confidence in a specific output.

---

## Connector

A connected data source attached to a Knowledge Stack. Connectors provide live or periodically synced data from external systems such as Slack, Jira, GitHub, Salesforce, Notion, or databases. Described by `AKSConnector` in the schema.

`sync` mode: data is periodically pulled and ingested into the compiled layer.
`live` mode: data is fetched at agent runtime and injected into context directly, bypassing compilation.

---

## Contributing documents

The set of source documents that contributed to a compiled entity's description via ingestion or cross-document consolidation. Tracked as `contributing_documents` (array of UUIDs) on `AKSEntity`. Grows as new documents are ingested. Used to generate sourced entity pages with inline citations.

---

## Domain

A lowercase hyphenated slug identifying the knowledge area of a Stack. Examples: `healthcare-billing`, `erp-integration`, `sales-enablement`. Used to resolve a Stack when no `stack_id` is known. Must be unique within a deployment.

---

## Domain context

The property of an entity description that reflects the entity's role within the full compiled domain, not just what one source document says about it in isolation.

An entity description has domain context when the compiler knew the full concept vocabulary before writing the description. A description of "Incident" written with domain context references Monitoring, On-Call, Rollback, and Postmortem if those entities exist in the same Stack, because the compiler knew they were there.

Domain context is established during the compilation pass. Implementations SHOULD survey all concepts across all source material before generating individual entity descriptions. This is distinct from cross-document consolidation, which merges multiple accounts of the same entity after the fact. Domain context means each entity was described with awareness of its neighbors from the start.

---

## Entity

A named concept in a Knowledge Stack, represented as a node in the knowledge graph. Every entity has an `id`, `name`, `type`, `description`, `properties`, `confidence`, and `source`. The `type` field is domain-defined; there is no global entity type enum.

The `description` field is the compiled account of this entity within the context of the full domain. It is not a summary of what one document says about the concept; it is a synthesized description that reflects the entity's role and relationships within everything the Stack knows. A well-compiled entity description reads like a wiki page written by someone with full domain knowledge, not like an extraction from a single source.

---

## Entity page

The human-readable representation of a compiled entity. Generated from the knowledge graph on demand. Never written manually. Always reflects the current compiled state of the graph. In v0.1.0, entity pages MAY also incorporate live connector data (hybrid pages). When `contributing_documents` is populated, entity pages include a Sources section with inline citations.

---

## Geometric mean scoring

A retrieval ranking approach that combines vector similarity and text (trigram) similarity using a geometric mean rather than treating each independently. A chunk only ranks highly when both signals agree; it must be both semantically close to the query and textually relevant. This prevents semantically adjacent but textually irrelevant chunks from surfacing, and prevents good textual matches from being suppressed due to embedding distance. Described in DECISIONS.md.

---

## Knowledge Base

A passive repository of documents that agents or users search against. A knowledge base stores content; it does not compile, structure, or reason over it. Searching returns text that looks similar to the query. It does not follow typed relationships, accumulate usage signal, surface its own gaps, or expose a structured query interface.

Examples: Confluence, Notion, Guru, a folder of PDFs with a search box.

A Knowledge Stack is not a knowledge base.

---

## Knowledge graph

The compiled structure of a Knowledge Stack: a directed graph where nodes are entities and edges are typed relationships between them. Not a graph database specifically; the term refers to the data structure, not the storage technology.

---

## Knowledge Stack

Compiled domain knowledge is a Knowledge Stack. When source material (documents, connected sources) is processed into a structured graph of entities and typed relationships, the result is a Knowledge Stack.

A Knowledge Stack is a live, compiled, versioned, queryable system. It is not a file. It is a running artifact with a `stack_id`, a `domain`, a `scope`, and a set of AKS-compliant endpoints. It accumulates run records. It accepts backfeed. It knows its own gaps. It gets smarter from usage.

A Knowledge Stack is distinct from an AKS Bundle. The Stack is the live system. The Bundle is a JSON snapshot produced on demand via `GET /aks/v1/export`.

---

## last_seen_at

A timestamp field on `AKSEntity` recording when this entity was most recently mentioned or consolidated by a new document ingestion. A staleness signal for Stack owners and consumers. Does not affect confidence automatically; the two fields are independent. See DECISIONS.md for why automatic temporal decay is not applied.

---

## Live layer

The portion of a Knowledge Stack that draws from connected sources at agent runtime rather than from the compiled ontology. Live data is not in the vector store and has not been through verification and consolidation. Runs and entity pages that incorporate live data carry `source_type: "live"` or `source_type: "hybrid"`.

---

## MCP (Model Context Protocol)

An open protocol that defines how AI agents connect to external tools and data sources. AKS endpoints MAY be wrapped as MCP tools so that development environments like Claude Code and Cursor can query a Knowledge Stack natively. MCP governs tool invocation. AKS governs the knowledge layer. They compose without overlap.

---

## Memory

Session-scoped, agent-specific context that tracks what a single agent has experienced. Memory is first-person and temporal: it belongs to one agent, organized around when things happened, and does not typically persist beyond the agent's lifetime.

Memory is raw input to a Knowledge Stack, not a substitute for one. Agent session outputs can be compiled into a Stack through the backfeed process. Once compiled, the resulting knowledge is available to any agent querying the Stack.

AKS is not a memory standard. It is the standard for the compiled layer that memory feeds into.

---

## Origin

The source of a backfeed submission.

- `run`: content produced by an agent run; use `source_run_id` to trace back
- `direct`: submitted without a run producing it

---

## Recency boost

A retrieval technique that gives a mild advantage to chunks from more recently ingested documents when similarity scores are close. Implemented as a soft multiplier that decays from 1.0 toward a floor value (e.g. 0.85) over a configurable window. Old content still surfaces; it just loses a small edge against fresh content. Applied at query time; does not modify stored data. Distinct from entity staleness signaling. See DECISIONS.md.

---

## Relationship

A typed directed connection between two entities; an edge in the knowledge graph. Every relationship has a `from_entity_id`, `to_entity_id`, a `type` expressed as a verb phrase, `confidence`, and `source`. Relationships MUST reference entity IDs rather than entity names.

---

## Run record

A log entry for a single agent run against a Knowledge Stack. Contains trigger type, query, traversal path, entities used, confidence, and `source_type` (compiled, live, or hybrid). Run records drive coverage awareness; low confidence runs on specific entities signal where the Stack needs more source material.

---

## Scope

The visibility boundary of a Knowledge Stack.

- `stack`: private to this Stack; default for all new Stacks
- `workspace`: shared within an organization; requires human review to graduate to
- `domain`: public and community-governed; requires threshold of verified entities and community approval

---

## Source

The traceability block on every entity and relationship. Records origin (`source_kind`, `document_id`, `connector_id`), when extracted (`extracted_at`), what model extracted it (`model`), whether a human reviewed it (`verified`), and which pipeline produced it (`pipeline`). The unit of trust evidence in AKS.

---

## source_kind

A field on `AKSSource` identifying how a knowledge unit was produced.

- `document`: extracted from an uploaded file
- `connector`: extracted from a connected source via sync
- `backfeed`: absorbed from an approved backfeed entry
- `manual`: written directly by a human reviewer

---

## source_type

A field on `AKSRun` and `AKSEntityPage` describing what knowledge sources contributed to the output.

- `compiled`: only the compiled ontology and vector store
- `live`: only live connector data fetched at runtime
- `hybrid`: both compiled and live sources

Consumers SHOULD use `source_type` to interpret confidence scores correctly.

---

## Stack composition

The practice of using multiple Knowledge Stacks together. Three patterns are supported:

`Import` absorbs one Stack's bundle into another through the standard compilation pipeline.
`Reference` binds an agent to a primary Stack while consulting others as read-only context.
`Federated query` has the consumer query multiple Stacks at runtime and merge results.

Source traceability is what makes composition tractable across Stack boundaries.

---

## Traversal path

The ordered list of entity names an agent followed when answering a query. Returned by `POST /aks/v1/context` and recorded in `AKSRun`. Makes the agent's reasoning explicit and auditable.

---

## Trigger

What initiated an agent run.

- `manual`: a person asked a question directly
- `auto`: the run fired without human initiation, via event or schedule
- `api`: called directly via the AKS API

---

## Verified

A boolean field on every entity and relationship source block. Defaults to `false` for machine-extracted knowledge. Set to `true` when a human has reviewed and confirmed the knowledge unit. The evidence for scope graduation.
