# Hierarchical Network

This document explains how BRIDGE merges chunk-level graphs into a unified brain-wide network through hierarchical linking and aggregation.

## The Boundary Problem

Core 2 produces independent graphs for each spatial chunk. But fibers don't respect chunk boundaries:

```
┌─────────────┬─────────────┐
│   Chunk A   │   Chunk B   │
│             │             │
│    ●────────┼────────●    │
│             │             │
│   (path 1)  │  (path 2)   │
│             │             │
└─────────────┴─────────────┘
              ↑
        Fiber crosses boundary,
        split into two paths
```

Core 3's job: reconnect paths that were artificially split.

## Hierarchical Merging Strategy

### Why Hierarchical?

A naive approach might check every boundary endpoint against every other. For N chunks with B boundary endpoints each, this is O(N²B²) — prohibitive for large volumes.

Hierarchical merging reduces this:
1. Group adjacent chunks (2³ = 8 at a time)
2. Link only within groups
3. Repeat with merged groups
4. Total complexity: O(N log N × B)

### The Octree Structure

Chunks are organized in an octree-like hierarchy:

```
Layer 0 (chunks):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│0,0│0,1│0,2│0,3│1,0│1,1│1,2│1,3│
└───┴───┴───┴───┴───┴───┴───┴───┘

Layer 1 (2×2×2 groups):
┌───────────┬───────────┐
│   0,0     │    0,1    │
└───────────┴───────────┘

Layer N (single component):
┌───────────────────────┐
│         0,0           │
└───────────────────────┘
```

Each layer halves the number of components (in 3D: ÷8).

## Linking Algorithm

### Step 1: Identify Boundary Endpoints

An endpoint is a "boundary endpoint" if its node belongs to a different chunk than its component:

```python
def find_search_endpoints(components):
    boundary_endpoints = []
    for comp in components:
        for endpoint in comp.endpoints:
            endpoint_chunk = get_chunk_id(endpoint)
            if endpoint_chunk not in comp.chunk_ids:
                boundary_endpoints.append((comp.id, endpoint))
    return boundary_endpoints
```

### Step 2: Find Candidate Links

For each boundary endpoint, find components that might contain matching paths:

```python
def find_linked_components(endpoint, chunk_map):
    endpoint_chunk = get_chunk_id(endpoint)
    candidate_component = chunk_map.get(endpoint_chunk)
    return candidate_component
```

### Step 3: Match Endpoint Pairs

Two endpoints link if they appear in each other's paths:

```
Component A:                Component B:
path: [..., ep_A]          path: [ep_B, ...]
                  ↓ match? ↓
                ep_A appears in B's path
                ep_B appears in A's path
                  ↓
              LINKED!
```

```python
def find_paired_endpoint(comp_A, comp_B, ep_A):
    # Check if ep_A appears in any path in comp_B
    for path in comp_B.paths:
        if ep_A in path:
            # Find the endpoint of this path
            ep_B = path[0] if path[-1] == ep_A else path[-1]
            return ep_B
    return None
```

### Step 4: Update Components

Matched endpoints are recorded in both components:

```python
comp_A.linked_components[comp_B.id][ep_A].append(ep_B)
comp_B.linked_components[comp_A.id][ep_B].append(ep_A)
```

## Nesting Components

### Path Merging

Linked components are merged by joining their paths:

```
Before merge:
  Comp A: path1 = [..., ep_A]
  Comp B: path2 = [ep_B, ...]
  Link: ep_A ↔ ep_B

After merge:
  Nested: path = [..., ep_A/ep_B, ...]
```

### Path Simplification

To keep memory bounded, merged paths are simplified:

```python
# Original path (all nodes)
path = ["n1", "n2", "n3", ..., "n100"]

# Simplified (endpoints + reference)
simplified = ["n1", "path_id_from_lower_layer", "n100"]
```

The full path can be reconstructed by recursively expanding references:

```python
def extract_full_path(record, component_dict):
    path = record["path"]
    expanded = []
    for item in path:
        if is_node(item):
            expanded.append(item)
        else:
            # Item is a path reference
            sub_path = expand_reference(item, component_dict)
            expanded.extend(sub_path)
    return expanded
```

### Preserving Scores

Path scores are aggregated during merging:

```python
merged_scores = {
    "FA": weighted_mean([path1.FA, path2.FA], [len1, len2]),
    "length": path1.length + path2.length,
    "alignment": weighted_mean([path1.alignment, path2.alignment]),
    # ...
}
```

## Processing Order

### Layer-by-Layer

Processing must be sequential between layers (dependencies):

```
Layer 0 → Layer 1 → Layer 2 → ... → Layer N
   ↑           ↑           ↑
   Must complete before next starts
```

### Within-Layer Parallelism

Within a layer, meta-chunks can be processed in parallel:

```python
# Layer 1 has multiple independent meta-chunks
for (x, y, z), indices in current_layer.items():
    # These can run in parallel
    process_meta_chunk(indices)
```

## Final Simplification

The top-level component contains all paths, but in simplified form. For analysis or export:

```python
# Load final network
network = load_component(top_level_id)

# Expand a path to full node sequence
for path_id, record in network.paths.items():
    full_path = extract_full_path(record, all_components)
    # full_path is now complete node list
```

## Edge Cases

### Isolated Components

Some chunk components may not link to any neighbors:
- Fibers that start and end within the chunk
- Very short fiber segments
- Noise/artifacts

These remain as separate paths in the final network.

### Multi-Way Junctions

A single endpoint might link to multiple components:

```
       Comp A
         │
         ↓
Comp B ←─┼─→ Comp C
         ↑
         │
       Comp D
```

These are handled by recording all links and letting path extraction resolve the topology.

## Why Not Direct Global Linking?

### Memory Constraints

Loading all chunk components simultaneously would exceed memory for large volumes.

### Locality of Reference

Most links are between adjacent chunks. Hierarchical grouping exploits this spatial locality.

### Incremental Processing

Hierarchical approach allows checkpointing and resumption:

```python
BRIDGE_setup(
    ...,
    start_layer=2,  # Resume from layer 2
)
```

## See Also

- [Pipeline Architecture](architecture.md) — Overall system design
- [Core 3 API](../reference/api/core3.md) — Function reference
- [Graph Class](../reference/api/graph.md) — Component data structures
