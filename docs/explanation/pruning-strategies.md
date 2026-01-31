# Pruning Strategies

This document explains the parameter-free graph reduction strategies used in BRIDGE.

## The Need for Pruning

Initial graph construction creates dense connectivity. Without pruning:
- Redundant paths obscure true fiber topology
- Memory usage scales poorly
- Path extraction becomes computationally expensive

## Minimum Spanning Forest

### Definition

A minimum spanning forest (MSF) is a collection of minimum spanning trees, one for each connected component. It provides:

- **Connectivity preservation** — All reachable nodes remain reachable
- **Redundancy removal** — Exactly one path between any two nodes
- **Optimality** — Minimizes total edge weight

### Edge Weighting

Edges are weighted by alignment score (inverted so lower = better):

```python
weight = 1.0 - alignment_score
# alignment_score: how well edge aligns with node orientations
```

### Algorithm

```python
# Kruskal's algorithm (simplified)
edges_sorted = sort_by_weight(edges)
forest = DisjointSet()

for edge in edges_sorted:
    if forest.find(edge.u) != forest.find(edge.v):
        forest.union(edge.u, edge.v)
        keep_edge(edge)
```

## Triangle Shortcut Removal

Before MSF, we remove "shortcuts" — edges that bypass better paths:

```
A ──────────── C    Remove if d(A,B) + d(B,C) ≈ d(A,C)
 ╲            ╱
  B ─────────
```

This preserves geometric fidelity while reducing edge count.

## Why Parameter-Free?

Traditional tractography requires thresholds:
- Angle threshold for continuation
- FA threshold for termination
- Curvature constraints

These require tuning per dataset.

MSF-based pruning is parameter-free:
- No thresholds to set
- Topology emerges from data
- Consistent behavior across datasets

<!-- Placeholder: Add more content on alternative pruning strategies -->

## See Also

- [Graph Construction](graph-construction.md)
- [Core 2 API](../reference/api/core2.md)
