# Knowledge Workbench Sync Contract v1

## 1. Purpose

Knowledge Workbench is a knowledge workflow in which Notion is used for thinking and editing, while GitHub provides a versioned representation of the resulting source material.

The synchronization layer exists to connect these two systems without turning the synchronization layer into a new source of truth.

The contract defines:

- source of truth
- synchronization direction
- object identity
- GitHub path mapping
- synchronization status
- idempotency
- failure semantics
- boundaries of the Gateway
- downstream processing boundaries

---

## 2. System Roles

### Notion

Notion is the source material and editing source.

It contains the author's original thinking, ongoing revisions, metadata, and editorial state.

Notion content must not be silently replaced by generated interpretations from the synchronization layer.

### GitHub

GitHub is the versioned representation of Knowledge Workbench source material.

Its primary purposes are:

- version history
- diff and review
- technical collaboration
- automation input
- traceable historical changes

GitHub Markdown is therefore a versioned representation, not an independent editorial source.

### Vercel Gateway

The Gateway is a stateless adapter between Notion and GitHub.

It is responsible for:

- receiving supported Notion webhook events
- identifying the affected Knowledge Workbench page
- reading the current Notion page
- converting Notion blocks into Markdown
- resolving the page's GitHub path through the manifest
- determining whether a GitHub write is necessary
- writing the Markdown representation to GitHub
- updating synchronization metadata in Notion

The Gateway does not become the source of truth.

The Gateway must not rewrite, summarize, reinterpret, or otherwise alter the author's thinking.

---

## 3. Data Architecture

There is one Knowledge Workbench data source.

The apparent Notion and GitHub Knowledge Workbench tables are two views of the same underlying Knowledge Workbench data source.

The conceptual architecture is:

    Knowledge Workbench
            |
            +-- Notion view
            |     = thinking / editing / source material
            |
            +-- GitHub representation
                  = versioned source material

The synchronization system must preserve this one-source model.

---

## 4. Synchronization Direction

The primary synchronization direction is:

    Notion
      |
      | webhook
      v
    Gateway
      |
      | GitHub API
      v
    GitHub

The Gateway currently implements Notion -> GitHub synchronization.

GitHub -> Notion is not part of Sync Contract v1.

GitHub changes must not be treated as an instruction to overwrite Notion content.

---

## 5. Object Identity

A Knowledge Workbench entry is identified by its Notion page ID.

The manifest maps the Notion page ID to its versioned GitHub representation.

Example:

    Notion page ID
        ->
    knowledge/research/<slug>.md

The manifest therefore acts as an explicit mapping layer rather than relying on title matching at synchronization time.

---

## 6. Manifest

The manifest records the objects that are allowed to participate in synchronization.

A page not present in the manifest is not a synchronizable Knowledge Workbench object.

For an unmapped page, the Gateway must:

- not write to GitHub
- not change the page to Pending
- return a clear unmapped result

The manifest is therefore a synchronization boundary, not a second content database.

---

## 7. GitHub Path

Each synchronized Knowledge Workbench entry has an explicit GitHub path.

The manifest is the Gateway's canonical machine-readable mapping from Notion page ID to GitHub path.

The GitHub Path property in Notion is synchronization metadata exposed to the editing and governance layer. It does not independently determine the write target in Sync Contract v1.

The Gateway must use the manifest when resolving the actual GitHub write path.

Current convention:

    knowledge/<kind>/<filename>.md

The GitHub path is part of the synchronization identity.

Changing a path is therefore a structural change and must be treated separately from ordinary content editing.

---

## 8. Synchronization Metadata

Knowledge Workbench contains synchronization metadata:

### GitHub Path

The target Markdown path in the knowledge-workbench repository.

### GitHub Sync Status

Current synchronization state.

Supported states:

- Pending
- Synced
- Outdated
- Conflict
- Error
- Disabled

The v1 data model defines Pending, Synced, Outdated, Conflict, Error, and Disabled as the available synchronization states.

The Gateway currently implements Pending, Synced, and Error.

Outdated, Conflict, and Disabled remain reserved states until their triggering conditions and ownership rules are explicitly defined.

### GitHub Last Synced

Timestamp of the most recent successful synchronization.

This records synchronization history metadata. It does not replace Git history.

---

## 9. Idempotency

Synchronization is content-idempotent.

Receiving the same effective content more than once must not create additional Git commits.

The Gateway therefore compares the desired Markdown representation with the existing GitHub file before writing.

If the existing GitHub content is identical:

    synced = true
    skipped = true

No GitHub write is performed.

This makes webhook delivery count independent from Git commit count.

A repeated webhook is therefore not expected to produce a repeated commit.

---

## 10. Webhook Semantics

The Gateway currently handles:

    page.content_updated

Other webhook event types are acknowledged but are not synchronization triggers in v1.

The webhook event identifies the page.

The Gateway then reads the current state from Notion rather than trusting the webhook payload as the complete content representation.

The webhook is therefore a change notification, not the source of content.

---

## 11. Current Reliability Boundary

The Gateway has verified:

- normal Notion -> GitHub synchronization
- repeated webhook idempotency
- content-level duplicate suppression
- unmapped-page handling
- near-simultaneous webhook behavior in a practical test

The system does not claim strict global event ordering.

If multiple different Notion versions are changed and delivered concurrently, v1 does not provide a durable event queue or historical event store.

Strict ordering therefore remains outside the current contract.

This is an explicit boundary rather than an implicit assumption.

---

## 12. Failure Semantics

A synchronization failure must not be represented as successful synchronization.

The Gateway attempts to update the Notion synchronization status to Error when a synchronization operation fails.

The GitHub repository remains the durable record of successfully committed representations.

A failed synchronization must therefore be distinguishable from:

- an unchanged file
- an unmapped page
- a successfully synchronized file

---

## 13. Source Preservation

The synchronization layer preserves source material.

The Gateway may transform Notion block structure into Markdown syntax, but it must not introduce semantic interpretation into the author's content.

Examples of transformations that belong to the adapter:

- Notion heading -> Markdown heading
- Notion paragraph -> Markdown paragraph
- Notion code block -> Markdown fenced code block
- Notion list -> Markdown list

Examples that do not belong to the adapter:

- rewriting an argument
- summarizing an essay
- changing the author's conclusion
- generating an Architecture Decision automatically
- converting research into atomic notes automatically

Those are downstream knowledge-work operations.

---

## 14. Downstream Knowledge Pipeline

Synchronization is only the first layer of the larger Knowledge Workbench workflow.

The intended workflow is:

    Original thinking
          |
          v
    Small additions of missing
    concepts / technical basis
          |
          v
    Architecture Decision
    or GitHub Issue
          |
          v
    Obsidian atomic knowledge

The synchronization layer should not collapse these stages into one automated transformation.

Each stage has a different purpose and therefore remains independently replaceable.

---

## 15. Architectural Principle

The core principle is:

> Notion负责创造，GitHub负责版本化，Gateway负责转换，Website负责呈现。

For Knowledge Workbench specifically:

> Notion负责思考与编辑，GitHub负责版本化原文，Gateway负责连接两者，而不是替作者思考。

The purpose of modularity here is not merely to split the system into more components.

It is to establish boundaries around change.

A component can be replaced when its external contract remains stable.

Therefore the long-term value of this architecture is not that Notion, GitHub, or Vercel must remain forever.

The value is that changing one implementation does not require rewriting the conceptual responsibility of every other layer.

---

## 16. Version

This document defines Knowledge Workbench Sync Contract v1.

Changes that alter source-of-truth rules, synchronization direction, object identity, or the meaning of synchronization metadata should be treated as contract changes rather than ordinary implementation details.
