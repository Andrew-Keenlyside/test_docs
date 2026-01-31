# Explanation

This section provides conceptual background and design rationale for BRIDGE. Unlike tutorials and how-to guides, these documents focus on *why* things work the way they do.

## Architecture & Design

| Topic | Description |
|-------|-------------|
| [Pipeline Architecture](architecture.md) | Overall system design and data flow |
| [Why Hierarchical Processing](hierarchical-network.md) | Rationale for multi-scale approach |

## Core Algorithms

| Topic | Description |
|-------|-------------|
| [Node Extraction](node-extraction.md) | Structure tensor and ODF-based detection |
| [Graph Construction](graph-construction.md) | Edge creation, segments, and MSF |
| [Pathfinding](pathfinding.md) | BFS-based path extraction and scoring |
| [Pruning Strategies](pruning-strategies.md) | Parameter-free graph reduction |

## Validation & Quality

| Topic | Description |
|-------|-------------|
| [Path Scoring](path-scoring.md) | Quality metrics and their interpretation |
| [Validation Approaches](validation.md) | How to assess reconstruction quality |

---

!!! info "Understanding vs. Doing"
    These documents explain concepts and reasoning. For step-by-step instructions, see [How-to Guides](../how-to/index.md). For API details, see [Reference](../reference/index.md).
