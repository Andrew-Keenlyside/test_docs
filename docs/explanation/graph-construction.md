# Graph Construction

This document explains how BRIDGE builds local graphs from extracted nodes, including edge creation, segment definition, and minimum spanning forest pruning.

## From Nodes to Graphs

After node extraction (Core 1), we have a point cloud of candidate fiber locations, each with:
- Position (z, y, x)
- Orientation (principal eigenvector)
- Quality metrics (FA, intensity, local z-score)

The challenge: connect these points into coherent fiber pathways without:
- Over-connecting (creating spurious shortcuts)
- Under-connecting (breaking true fibers)
- Requiring manual threshold tuning

## Edge Creation

### Spatial Proximity

Initial edges connect nodes within a maximum distance:

```python
tree = KDTree(positions)
edges = tree.query_pairs(r=max_edge_distance)
```

This creates a dense initial graph. A chunk with 10,000 nodes might have 500,000 candidate edges.

### Alignment Filtering

Not all nearby nodes should connect. We filter by geometric alignment:

```
              edge vector
       n1 ──────────────────► n2
        ↑                      ↑
        │ v1              v2   │
        │                      │
   eigenvector            eigenvector
```

An edge is valid if both node eigenvectors align with the edge direction:

```python
angle_1 = arccos(|dot(v1, edge_direction)|)
angle_2 = arccos(|dot(v2, edge_direction)|)
valid = (angle_1 < limit) and (angle_2 < limit)
```

The absolute value handles eigenvector sign ambiguity (v and -v represent the same orientation).

### Triangle Shortcut Removal

Dense graphs contain many shortcuts — direct edges that bypass longer but more accurate paths:

```
      A ─────────────── C    (shortcut)
       ╲               ╱
        ╲             ╱
         ╲           ╱
          B ────────        (better path)
```

We remove edge A-C if there exists B such that:
```
d(A,B) + d(B,C) ≤ (1 + ε) × d(A,C)
```

This preserves topological connectivity while removing redundant shortcuts.

## Segment Definition

### What is a Segment?

A segment is a node triplet (a, b, c) representing a local fiber direction:

```
    a ────── b ────── c
         segment
```

Segments are the atomic units of fiber reconstruction. They capture:
- Local continuity (three connected nodes)
- Directional flow (a → b → c ordering)
- Curvature constraints (maximum bend angle)

### Segment Filtering

Valid segments must satisfy:
1. Edges a-b and b-c exist
2. Total span ≤ segment_radius
3. Turn angle at b ≤ segment_max_angle

The turn angle constraint prevents sharp bends:

```
           b
          ╱ ╲
         ╱ θ ╲
        a     c

valid if θ ≤ segment_max_angle
```

### Segment Graph

Segments form their own graph where:
- Nodes = segments
- Edges = overlapping segments (sharing two nodes)

Two segments connect if they share an edge:

```
Segment 1: (a, b, c)
Segment 2: (b, c, d)
             ↑↑
         shared edge

Segments connect: seg1 → seg2
```

This creates a directed graph of possible fiber continuations.

## Minimum Spanning Forest

### The Pruning Problem

The segment graph is still too dense. Many segments represent the same underlying fiber, creating redundant paths. We need principled pruning.

### Why MSF?

Minimum spanning forest (MSF) is a parameter-free graph reduction:

1. **Connectivity preserved** — All originally connected nodes remain reachable
2. **Redundancy removed** — Exactly one path between any two points
3. **Quality maximized** — Prefers edges with better alignment scores

```
Before MSF:              After MSF:
    ┌───┐                    ┌───┐
    │   │                    │   │
A───B───C              A───B   C
    │   │                      │
    └───┘                      │
    │                          │
    D                          D
```

### Implementation

```python
# Compute edge weights (lower = better)
weights = 1 - alignment_scores  # alignment in [0, 1]

# Build undirected graph
G = cugraph.Graph()
G.from_cudf_edgelist(edges, weights='weight')

# Find minimum spanning forest
mst_edges = cugraph.minimum_spanning_tree(G)
```

### State-Aware MSF

Standard MSF treats all edges equally. BRIDGE extends this with state awareness:

- **Endpoint preference** — Seeds from identified endpoints
- **Component size filtering** — Remove tiny disconnected components
- **Directionality** — Maintain segment flow direction

## Path Extraction

### BFS from Seeds

After MSF pruning, we extract paths using breadth-first search:

1. Identify seed segments (at component boundaries or density peaks)
2. BFS outward from seeds through segment graph
3. Collect node sequences along traversed segments
4. Score and filter resulting paths

```python
def bfs_paths(graph, seeds, max_depth):
    paths = []
    for seed in seeds:
        visited = set()
        queue = [(seed, [seed])]
        while queue:
            current, path = queue.pop(0)
            if len(path) > max_depth:
                continue
            for neighbor in graph.neighbors(current):
                if neighbor not in visited:
                    visited.add(neighbor)
                    new_path = path + [neighbor]
                    queue.append((neighbor, new_path))
                    if is_endpoint(neighbor):
                        paths.append(new_path)
    return paths
```

### Path Scoring

Each path receives quality scores:

| Score | Computation | Meaning |
|-------|-------------|---------|
| FA | Mean of node FA values | Fiber coherence |
| Alignment | Mean edge-vector angle | Geometric consistency |
| Path distance | Sum of edge lengths | Total fiber length |
| Bending angle | Mean turn angles | Curvature |
| Intensity | Mean node intensities | Signal strength |

## Why This Approach?

### Compared to Streamline Tractography

Traditional dMRI tractography:
- Follows continuous ODF/tensor field
- Requires step size, angle threshold, stopping criteria
- Sensitive to parameter choices

BRIDGE graph approach:
- Discrete nodes connected by geometric constraints
- MSF provides automatic pruning
- Parameters have physical meaning (distances, angles)

### Compared to Deep Learning

Neural network approaches:
- Require training data
- Black-box decisions
- May not generalize across imaging modalities

BRIDGE geometric approach:
- No training needed
- Interpretable constraints
- Applies to any oriented 3D data

## See Also

- [Pruning Strategies](pruning-strategies.md) — More on MSF and alternatives
- [Pathfinding](pathfinding.md) — BFS details and optimizations
- [Core 2 API](../reference/api/core2.md) — Function reference
