# Roadmap

This document describes what AKS does not yet cover and what is being considered for future versions. The current release is v0.1.0, documented in `README.md`.

Items listed here are directional, not committed. Priorities will shift based on implementation experience, community feedback, and what adopters actually need in production.

---

## Under Consideration

### Formal connector sync protocol

The current spec defines what a connector looks like as a data object (`AKSConnector` with `sync` and `live` modes) but does not specify how implementations should schedule syncs, handle partial failures, or resolve conflicts when a connector re-syncs content that has drifted. Production deployments will surface what this protocol needs to look like.

### Coverage score as a bundle-level field

A computed score reflecting how well-covered the Stack's entities are, based on run history, verification status, and contributing document counts. Makes Stack health a first-class metric in the bundle rather than something consumers compute themselves.

### Consolidation count on entities

A `consolidation_count` field on `AKSEntity` recording how many distinct source documents contributed to this entity's description. Provides a strength signal orthogonal to confidence. An entity consolidated from fifteen sources is different from one extracted from a single chunk, even if both report high confidence.

### Formal Stack Import endpoint

The Import composition pattern currently reuses the ingestion pipeline. A dedicated `POST /stacks/import` endpoint would formalize this path, allowing implementations to handle bundle-level import semantics (entity deduplication, relationship merging, source provenance preservation) more cleanly than treating the bundle as a generic upload.

### Entity page frontmatter recommendation

Entity pages are markdown but have no standardized frontmatter. Recommending a YAML frontmatter structure (entity_id, name, type, confidence, generated_at, graph_version) would make entity pages interoperable with static site generators and knowledge management tools without requiring a parser change.

### Formal flow step schema

An `AKSFlowStep` schema for multi-step agent workflows that operate against a Stack. Captures the structure of how an agent reasons through a query beyond what `traversal_path` alone can express.

---

## Open Questions

Questions the spec does not yet take a position on but likely will in future versions:

- Should AKS define a standard way to expose Stack metadata (last compiled, entity count trends, active connectors) for dashboards and monitoring?
- Should the spec recommend or require authentication semantics for the AKS endpoints, or remain silent and let implementations choose?
- Should there be a standardized way to deprecate entities, beyond deleting them, that preserves run record references?
- How should implementations handle the case where a document is deleted but contributed to an entity that has since been verified?

These are not blockers for v0.1.0 but are worth resolving before implementations diverge on their own answers.

---

## Non-Goals

The following are deliberately not on the roadmap. Naming them explicitly to prevent scope creep:

- **Agent definition.** How agents consume Knowledge Stacks is out of scope. AKS governs the knowledge layer, not the agent layer.
- **Tool invocation.** MCP already covers this surface. AKS integrates with MCP rather than duplicating it.
- **Authentication and authorization.** Implementations choose this. 
- **A reference UI.** The spec describes the data and endpoints, not how they should be visualized. Implementations build their own interfaces.
- **Storage technology.** Postgres and pgvector happen to be what the reference implementation uses. Any storage backend capable of serving the endpoints is conformant.

---

## Contributing to the Roadmap

If you are building on AKS and find yourself working around a gap in the spec, that gap probably belongs here. Open an issue describing the use case and the workaround. Real production constraints drive better spec decisions than anticipated ones.