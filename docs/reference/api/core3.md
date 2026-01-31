# Core 3: Hierarchical Network

Core 3 merges chunk-level graphs into a unified brain-wide network through hierarchical linking and aggregation.

## Main Entry Point

### `process_hierarchical_network`

```python
from bridge.core.core3_hierarchical_network import process_hierarchical_network

process_hierarchical_network(
    directory: str,
    sub_id: str = "01",
    res: str = "1",
    start_layer: int = None,
    overwrite: bool = True
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `directory` | str | - | BRIDGE output directory |
| `sub_id` | str | "01" | Subject identifier |
| `res` | str | "1" | Resolution string |
| `start_layer` | int | None | Resume from specific layer |
| `overwrite` | bool | True | Overwrite existing components |

## Processing Pipeline

```
Layer 0 Components (chunk-level)
    ↓
[find_search_endpoints] → Boundary endpoints
    ↓
[find_linked_components_from_endpoint] → Cross-boundary links
    ↓
[find_paired_endpoint] → Matched endpoint pairs
    ↓
[link_components] → Update linked_components
    ↓
[nest_components] → Merged component
    ↓
Repeat for each layer until single component
    ↓
[simplify_and_save_final] → Final unified network
```

## Key Functions

### `find_search_endpoints`

Identify endpoints that extend beyond component boundaries.

```python
search_endpoints, chunk_to_component_map = find_search_endpoints(
    search_components: list  # Components at current layer
)
```

**Returns:**
- `search_endpoints`: List of (component_id, endpoint_id) tuples
- `chunk_to_component_map`: Dict mapping chunk_id to component_id

### `find_linked_components_from_endpoint`

Find components that share an endpoint.

```python
component_id, links = find_linked_components_from_endpoint(
    component_id: str,
    endpoint_id: str,
    chunk_to_component_map: dict
)
```

**Returns:**
- `links`: Dict of `{found_component_id: [endpoint_ids]}`

### `find_paired_endpoint`

Find matching endpoints in linked components.

```python
endpoint_2_ids = find_paired_endpoint(
    component_1,
    component_2,
    endpoint_1_id: str,
    linked_endpoints: dict,
    component_dict: dict
)
```

### `link_components`

Update component linking information.

```python
result = link_components(
    search_id,
    found_id,
    endpoint,
    search_component,
    found_component,
    linked_endpoints,
    component_dict
)
```

**Returns:**
```python
{
    "search_id": str,
    "found_id": str,
    "endpoint": str,
    "paired_endpoints": list
}
```

### `extract_path`

Recursively resolve a path through the hierarchy.

```python
path = extract_path(record, component_dict)
# Expands simplified paths (endpoint, path_id, endpoint)
# to full node sequences
```

### `nest_components`

Merge a group of linked components into a parent component.

```python
nested = nest_components(
    component_ids: set,
    directory: str,
    graphs_dir: str,
    parent_id: str,
    component_dict: dict,
    layer_i: int
)
```

**Returns:** New Graph component containing merged paths

### `define_layers`

Build hierarchical layer structure from chunk indices.

```python
layers = define_layers(base_dir)
# layers[0] = {(0,0,0): [(0,0,0)], (0,0,1): [(0,0,1)], ...}
# layers[1] = {(0,0,0): [(0,0,0), (0,0,1), (0,1,0), (0,1,1)], ...}
# ...
```

## Hierarchy Structure

Each layer groups 2³ = 8 components from the previous layer:

```
Layer 0: Individual chunks
         ┌─────┬─────┬─────┬─────┐
         │0-0-0│0-0-1│0-1-0│0-1-1│ ...
         └─────┴─────┴─────┴─────┘

Layer 1: 2×2×2 groups
         ┌───────────┬───────────┐
         │   0-0-0   │   0-0-1   │ ...
         └───────────┴───────────┘

Layer N: Single unified network
         ┌───────────────────────┐
         │         0-0-0         │
         └───────────────────────┘
```

## Path Simplification

Higher-layer components store simplified paths:

```python
# Layer 0 (full path)
path = ["node_1", "node_2", "node_3", ..., "node_100"]

# Layer 1+ (simplified)
path = ["node_1", "path_id_from_layer_0", "node_100"]
```

Full paths are reconstructed on demand via `extract_path()`.

## CPU Execution

Core 3 runs on CPU using:

| Library | Usage |
|---------|-------|
| NetworkX | Graph operations |
| GraphBLAS | Sparse matrix algorithms |
| Dask | Distributed task scheduling |

## Output

Final network is saved to `{directory}/network/hierarchy/lvl_{N}/`:

```
sub-{sub_id}_res-{res}mm_lvl-{N}_cid-0-0-0.zarr/
├── .zattrs          # Full network metadata
├── paths/           # All paths (simplified)
├── endpoints/       # All endpoints
└── ...
```

## See Also

- [Graph Class](graph.md)
- [Core 2: Local Graphs](core2.md)
- [Hierarchical Network Explained](../../explanation/hierarchical-network.md)
