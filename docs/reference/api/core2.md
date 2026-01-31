# Core 2: Local Graph Construction

Core 2 builds local graphs within each chunk by connecting nodes based on spatial proximity and geometric constraints.

## Main Entry Point

### `build_local_graphs`

```python
from bridge.core.core2_local_graphs import build_local_graphs

build_local_graphs(
    directory: str,
    max_edge_distance: float = 60,
    edge_allignment_limit: float = 60,
    segment_radius: float = 45,
    sub_id: str = "01",
    res: str = "1",
    overwrite: bool = True
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `directory` | str | - | BRIDGE output directory |
| `max_edge_distance` | float | 60 | Maximum distance for node connections (µm) |
| `edge_allignment_limit` | float | 60 | Maximum angle between edge and node vectors (°) |
| `segment_radius` | float | 45 | Radius for segment grouping |
| `sub_id` | str | "01" | Subject identifier |
| `res` | str | "1" | Resolution string |
| `overwrite` | bool | True | Overwrite existing components |

## Processing Pipeline

```
Node Files
    ↓
[load_chunk_and_overlap] → Nodes with neighboring chunks
    ↓
[build_graph_from_nodes] → Initial edge list (KD-tree)
    ↓
[refine_graph_edges] → Filtered edges (alignment, shortcuts)
    ↓
[define_segments] → Node triplets (a-b-c)
    ↓
[build_segment_graph] → Segment connectivity
    ↓
[minimum_spanning_forest] → Pruned segment graph
    ↓
[search_segment_paths] → Path extraction (BFS)
    ↓
[score_paths] → Path quality metrics
    ↓
[save_component_zarr_like] → Graph component files
```

## Key Functions

### `build_graph_from_nodes`

Build initial graph using spatial proximity.

```python
node_df, node_dict, node_ids, edge_df, id_to_int, int_to_id = build_graph_from_nodes(
    nodes: list,      # List of Node objects
    radius: float     # Maximum edge distance
)
```

**Returns:**
- `node_df`: cuDF DataFrame with node positions and vectors
- `edge_df`: cuDF DataFrame with src, dst, distance columns
- `id_to_int`, `int_to_id`: ID mapping dictionaries

### `refine_graph_edges`

Filter edges by alignment and remove triangle shortcuts.

```python
edge_df = refine_graph_edges(
    node_df,
    edge_df,
    edge_alignment_limit: float,     # Max angle in degrees
    triangle_max_neighbors: int = 12,
    triangle_eps_dist: float = 0.1
)
```

### `triangle_shortcut_dedup`

Remove shortcut edges where two-hop path is not longer than direct edge.

```python
edge_df = triangle_shortcut_dedup(
    edge_df,
    max_neighbors_per_node: int = 12,
    eps_dist: float = 0.05
)
```

### `define_segments`

Create segments (node triplets) from edges.

```python
seg_df, kept_segments_dict, centres = define_segments(
    edge_df,
    nodes,
    node_ids,
    id_to_int,
    int_to_id,
    min_radius: float
)
```

A segment is a triplet (a, b, c) where edges a-b and b-c exist.

### `minimum_spanning_forest_state_aware`

Prune segment graph using minimum spanning forest.

```python
seg_edges_filtered = minimum_spanning_forest_state_aware(
    seg_edges,
    seg_df,
    min_component_size: int = 5
)
```

### `search_segment_paths`

Extract paths through BFS from seed segments.

```python
all_node_paths, node_ids, node_idxs = search_segment_paths(
    segG,                    # cuGraph segment graph
    kept_segments_dict,      # Segment definitions
    int_to_id,               # ID mapping
    centres,                 # Segment centres
    seg_df,                  # Segment DataFrame
    seeds: list,             # Seed segment IDs
    min_vertices: int = 3,
    min_len: int = 5,
    max_depth: int = 100,
    seed_batch_size: int = 1000
)
```

### `score_paths`

Compute quality metrics for extracted paths.

```python
records, start_paths, end_paths, path_endpoints = score_paths(
    all_node_paths,
    node_attrs,    # GPU array of [img, fa, local_z]
    adj_df,        # Edge attributes
    seg_df,        # Segment attributes
    int_to_id,
    cid: str,
    res: float
)
```

**Computed Scores:**
- `FA`: Mean fractional anisotropy
- `intensity`: Mean image intensity
- `local_z`: Mean local z-score
- `alignment`: Mean edge-vector alignment
- `path_distance`: Total path length
- `edge_distance`: Mean edge length
- `bending_angle`: Mean turn angle
- `curve_radius`: Estimated curvature radius

## GPU Acceleration

Core 2 uses RAPIDS libraries:

| Library | Usage |
|---------|-------|
| cuDF | DataFrames for nodes, edges, segments |
| cuGraph | Graph algorithms (MSF, BFS, connected components) |
| CuPy | Array operations, spatial computations |
| cuSparse | Sparse matrix operations |

## Output

Components are saved to `{directory}/network/hierarchy/lvl_0/`:

```
sub-{sub_id}_res-{res}mm_lvl-0_cid-{chunk_id}.zarr/
├── .zattrs
├── paths/
├── nodes/
└── ...
```

## See Also

- [Graph Class](graph.md)
- [Core 3: Hierarchical Network](core3.md)
- [Graph Construction Explained](../../explanation/graph-construction.md)
