# Design Decisions

This document explains why AKS is designed the way it is. When the spec makes a choice that might seem arbitrary or restrictive, the reasoning is here.

---

## Why humans are in the loop at every critical step

AKS draws heavily from the knowledge management discipline, which has spent decades working out how organizations actually build, verify, and maintain their institutional knowledge. One of the most consistent findings from that field is that knowledge is not trustworthy unless a human has taken responsibility for it. Automated systems can extract, organize, and retrieve information, but the judgment of whether a given piece of information is correct, current, and appropriate for a given context has not been successfully automated in any meaningful way.

This is why human involvement appears throughout the spec as a first-class concept rather than an afterthought. The `source.verified` flag defaults to `false` for machine-extracted knowledge and only flips to `true` after human review. Scope graduation from `stack` to `workspace` to `domain` is gated on verified entity thresholds, not on model confidence scores. Backfeed content cannot enter the compiled knowledge without human approval, regardless of how confident the agent that produced it was. Finally, document audits surface recommendations to a human before ingestion rather than proceeding silently.

None of these gates exist because machine extraction is unreliable in principle. They exist because a Knowledge Stack is institutional knowledge, and institutional knowledge requires institutional accountability. A wrong entity extracted by an LLM and never reviewed is a wrong entity anyone on the team will encounter and any AI system might reason from. The human in the loop is what turns a pile of extracted content into something an organization can actually stand behind.

This design choice comes with real friction. Teams with large document backlogs cannot flip every extracted entity to verified on day one. The spec acknowledges this by making `verified` optional and graduation voluntary: a Stack can operate indefinitely at `stack` scope with mostly unverified entities. What it cannot do is graduate to shared scope without the verification work having been done. The spec's position is that this friction is the point, not a limitation. Shortcutting it produces Stacks that no one trusts, which is worse than having no Stack at all.

---

## Why relationships reference entity IDs not names

Entity names change. A team might rename "Customer" to "Account" when their product pivots. If relationships stored names rather than IDs, every relationship involving that entity would break silently. By referencing UUIDs, the relationship survives a rename. The consumer resolves display names from the entities array at read time.

---

## Why document content is not in the exported Stack bundle

A document corpus can be gigabytes. A compiled entities and relationships of the same corpus is kilobytes. The exported bundle is designed to be portable: importable by any tool, queryable immediately, shippable over an API. The bundle carries enough to reconstruct trust (which documents were used, how good they were, which entities they produced) without the raw payload.

---

## Why entity and relationship types are domain-defined with no global enum

A healthcare billing Stack has entity types like Procedure Code, Payer, and Authorization Request. An ERP Stack has Cost Code, Contract, and Purchase Order. An ecology Stack has Organism Role, Abiotic Component, and Trophic Level. These vocabularies have nothing in common. A global enum would either be impossibly large or uselessly generic.

---

## Why entity descriptions require domain context

An entity description written in isolation and extracted from a single document chunk without knowledge of what other entities exist in the Stack produces a shallow compiled artifact. It may accurately describe what one source said about the concept, but it cannot describe the concept's role within the domain because the compiler didn't know the domain when it wrote the description.

Domain context means the compiler knew the full vocabulary before writing any individual description. When the compilation pass knows that a software engineering Stack contains PR, Code Review, CI Pipeline, Deployment, Incident, and Postmortem, its description of Incident can reference Monitoring (which detects it), On-Call (which it triggers), Rollback (which resolves it), and Postmortem (which it produces); because those concepts were already established before the description was written.

The `description` field SHOULD NOT be the raw extraction output of a single LLM call against a single chunk. A two-phase compilation approach produces better descriptions than a document-by-document sequential approach. Phase one establishes the concept vocabulary across all sources. Phase two generates each entity description with the full vocabulary in context. The output format is identical (AKS-compliant entities and relationships); the quality of the descriptions is significantly higher.

---

## Why entity pages are generated on demand and never stored

A stored entity page goes stale the moment the graph changes. If a new document is ingested and three entities are updated, stored pages for those entities are now wrong; but not visibly so. Generating on demand means every page reflects the current compiled state.

Implementations MAY cache pages for performance but MUST invalidate the cache whenever the underlying entity or its relationships change. The `graph_version` field provides a standard mechanism for this.

---

## Why entity pages can incorporate live connector data

A compiled entity page for "Incident" captures structural knowledge from ingested postmortems and runbooks. A person reading an entity page for "Incident" in an operational context also wants to know: what incidents are open right now? Live connector data addresses this. The `source_type` field distinguishes compiled pages from hybrid ones. Consumers who require only verified compiled knowledge filter accordingly.

---

## Why entity pages can be fully reconstructed from a Bundle

This is a deliberate design guarantee. An AKS Bundle is self-contained for consumption purposes. A team that exports a Bundle can import it into any AKS-compliant implementation and immediately generate entity pages with no dependency on the original source documents. The Bundle contains all entities, all relationships, all confidence scores, and all source traceability; everything the synthesis step needs.

If source documents are also available, the compilation pipeline can be re-run for potentially richer output. But the Bundle alone should be sufficient for AI consumption.

---

## Why source_hash, truncated, and original_length are on AKSDocument

**The recompilation efficiency problem.** Without `source_hash`, an implementation has no standard way to detect whether a re-ingested document has actually changed. `source_hash` standardizes the signal so any AKS-aware implementation can compute it and skip redundant extraction passes.

**The truncation honesty problem.** Long documents routinely exceed extraction model context windows and get silently truncated. The compiled entities look identical to entities compiled from complete sources. `truncated: true` with `original_length` makes this visible. A Stack owner can see which documents were truncated and take action.

---

## Why last_seen_at is on AKSEntity rather than decaying confidence automatically

A Knowledge Stack is compiled domain knowledge, not agent memory. The entity "Incident" defined in a runbook does not become less true because nobody queried it last month. Automatically decaying entity confidence would corrupt the compiled ontology; high-confidence entities would quietly erode with no action on the Stack owner's part.

`last_seen_at` surfaces staleness as information without applying it automatically. The Stack owner decides whether an entity that has not appeared in new sources for 90 days warrants review, re-ingestion, or no action. The field provides the signal; the policy is the owner's to set.

This is distinct from `confidence`, which measures extraction quality. An entity can have high confidence and be stale, or low confidence and be current. They are independent signals and MUST remain separate fields.

Automatic temporal decay belongs in agent memory systems (like BrainDB's decay model). Knowledge Stack entities represent compiled domain truth, not accessed memories. The staleness signal is appropriate; the automatic decay is not.

---

## Why geometric mean is recommended for retrieval scoring

Pure cosine similarity can fail in two specific cases - 
- High vector similarity, low text relevance: semantically adjacent concepts surface for queries where they are not actually useful. 
- Low vector similarity, high text relevance: domain-specific terminology with low embedding alignment gets suppressed even when the chunk is exactly what the user needs.

Geometric mean combines both signals with a penalizing effect when one signal is weak. A chunk only ranks highly when both its semantic meaning and its textual content agree with the query. This produces more reliable retrieval without introducing additional LLM calls.

The short-query caveat matters: trigram similarity scores poorly on queries shorter than three words due to insufficient n-gram overlap. Implementations SHOULD apply a neutral text score (0.5) when trigram similarity falls below a threshold, rather than pulling the geometric mean toward zero and suppressing otherwise valid results.

---

## Why recency weighting is recommended but not automatic staleness decay

Retrieval recency weighting and entity staleness signaling solve different problems.

**Retrieval recency weighting** applies at query time, not at storage time. When two chunks have similar similarity scores, prefer the one from a more recently ingested document. This is appropriate because newer information is more likely to reflect current reality. The decay is soft; old content still surfaces, it just loses a mild edge. This does not modify any stored data.

**Entity staleness signaling** surfaces the `last_seen_at` timestamp to owners and consumers. It does not modify confidence scores, does not re-rank entities in responses, and does not remove anything from the ontology. It is information, not an automated action.

The two are deliberately separated. One affects query-time ranking. The other surfaces ownership information. Conflating them would produce a system where old-but-true knowledge silently fades in reliability, which is incorrect for a knowledge infrastructure system.

---

## Why compilation_strategy is a bundle-level field

The difference between incremental and global compilation produces materially different entity description quality. Consumers of a bundle; particularly those using Path 3 (Bundle in Claude Projects); should be able to tell whether the entities they are about to work with have domain-aware descriptions or per-document isolated ones.

The field surfaces this signal without mandating how an implementation chooses to compile. An implementation that does not track this MAY omit the field (it is OPTIONAL), in which case consumers should assume the lower-quality incremental path.

---

## Why the spec governs the output layer not the compilation pipeline

How knowledge is compiled (what chunking strategy, what extraction prompt, what confidence threshold) is where competitive differentiation lives. AKS defines what compiled knowledge looks like when it leaves a system. It says nothing about how that knowledge was produced. This is the same boundary HTTP draws: the protocol defines the message format, not what your server does to generate the response.

---

## Why scope is three levels rather than binary

Binary (private/public) does not reflect how knowledge actually moves through organizations. A team compiles knowledge useful to other teams before it is mature enough to publish publicly. The workspace scope captures this intermediate state.

---

## Why backfeed requires human review before absorption

Machine-extracted knowledge starts unverified and backfeed content is machine-produced. Automatically absorbing it would compound errors: a wrong answer, flagged and auto-absorbed, becomes a permanent wrong belief. Human review is the quality gate. This is what distinguishes AKS from memory systems that auto-save agent outputs.

---

## Why `source.verified` is the unit of graduation evidence rather than confidence score

Confidence scores are set by extraction pipelines and can be arbitrarily high from poorly calibrated models. Human verification cannot be faked; it requires a person to have reviewed and confirmed the knowledge unit. Graduation based on `verified` means a Stack graduates based on demonstrated human oversight, not on a model's self-assessment.

---

## Why Stack composition doesn't require new spec surface

Three composition patterns exist in practice (Import, Reference, Federated Query). Each is implementable with the existing spec surface without new endpoints or schema changes. Import uses the bundle format. Reference and Federated Query use the standard query endpoints. The spec's job is to make any two conformant Stacks speak the same query language. Composition emerges from that without the spec needing to prescribe a topology.

---

## Why Knowledge Stack and not Knowledge Base

A knowledge base is passive storage. You put documents in. You search them. The system returns text that looks similar to your query. It does not understand what it contains, does not know what questions have been asked, and does not get smarter from usage.

A Knowledge Stack is active infrastructure. It compiles documents into structured understanding, tracks every agent run, accumulates coverage signal, accepts backfeed, and exposes a standard query interface. The word "stack" describes distinct layers (source material, compiled ontology, query interface, entity pages) where each is produced from the one below it. Using "knowledge base" would invite the assumption that this is a search index with a fancy API. It is not.

---

## Why Knowledge Stack and AKS Bundle are named differently

A Knowledge Stack is a live system with endpoints. An AKS Bundle is a portable JSON file. The distinction matters because the spec places different requirements on each. "The Stack MUST expose..." applies to the live system. "The bundle MUST include..." applies to the exported file. Conflating the two produces implementations that satisfy one requirement while missing the other.

---

## Why the spec does not include tool or MCP definitions

MCP already has a spec. Adding tool definitions to AKS would make AKS partially redundant with it. AKS owns compiled knowledge, MCP owns tool invocation, and they compose cleanly without overlap. The MCP Integration section in the README is guidance, not a spec requirement.
