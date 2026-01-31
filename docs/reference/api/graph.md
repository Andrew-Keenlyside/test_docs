# Graph Class

The `Graph` class represents a network component in the BRIDGE hierarchy. It stores paths, endpoints, and connectivity information.

## Class Definition

```python
from bridge.objs.Graph import Graph
```

## Attributes

### Identity

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | str | Component identifier (BIDS format) |
| `layer` | int | Hierarchy level (0 = chunk level) |
| `lowest_layer` | bool | True if contains nodes, False if contains subcomponents |

### Node Data

| Attribute | Type | Description |
|-----------|------|-------------|
| `nodes` | set | Node IDs in this component |
| `endpoints` | set | Node IDs that are path endpoints |
| `unsearched` | set | Nodes not yet visited in search |
| `node_labels` | dict | Segmentation labels: `{node_id: label}` |

### Path Data

| Attribute | Type | Description |
|-----------|------|-------------|
| `paths` | dict | Path records: `{path_id: record}` |
| `start_paths` | dict | Paths by start node: `{node_id: [path_ids]}` |
| `end_paths` | dict | Paths by end node: `{node_id: [path_ids]}` |
| `path_endpoints` | dict | Endpoint pairs: `{path_id: (start, end)}` |

### Hierarchy

| Attribute | Type | Description |
|-----------|------|-------------|
| `subcomponents` | set | Direct child component IDs |
| `linked_components` | dict | Cross-boundary links |
| `chunk_ids` | set | Chunk identifiers covered |

### Spatial

| Attribute | Type | Description |
|-----------|------|-------------|
| `centre_coords` | np.ndarray | Mean spatial centre |
| `adj_matrix` | np.ndarray | Adjacency matrix (optional) |

## Path Record Structure

Each path in `self.paths` is a dictionary:

```python
{
    "path": ["node_id_1", "node_id_2", ..., "node_id_n"],
    "length": 42,
    "scores": {
        "FA": 0.45,
        "intensity": 180,
        "local_z": 1.2,
        "alignment": 15.3,
        "path_distance": 120.5,
        "edge_distance": 8.2,
        "bending_angle": 12.0,
        "curve_radius": 45.0
    }
}
```

## Methods

### `__init__(self, id=None)`

Create a new Graph instance.

```python
graph = Graph(id="sub-01_res-0.02mm_lvl-0_cid-0-0-0")
```

### `simplify(self) -> Graph`

Reduce paths to endpoint triples for hierarchical storage.

```python
# Before: path = ["n1", "n2", "n3", "n4", "n5"]
graph.simplify()
# After: path = ["n1", path_id, "n5"]
```

### `to_dict(self) -> dict`

Convert graph to dictionary for JSON/Zarr serialization.

```python
d = graph.to_dict()
```

### `from_dict(cls, d: dict) -> Graph`

Create graph from dictionary.

```python
graph = Graph.from_dict(d)
```

## Usage Examples

### Loading a Component

```python
from bridge.utils.utils_components import load_component

# Load by ID
graph = load_component(
    "sub-01_res-0.02mm_lvl-0_cid-0-0-0",
    directory="network/hierarchy"
)

print(f"Component: {graph.id}")
print(f"Nodes: {len(graph.nodes)}")
print(f"Paths: {len(graph.paths)}")
print(f"Endpoints: {len(graph.endpoints)}")
```

### Iterating Paths

```python
for path_id, record in graph.paths.items():
    path = record["path"]
    scores = record.get("scores", {})
    
    print(f"Path {path_id}:")
    print(f"  Length: {len(path)} nodes")
    print(f"  FA: {scores.get('FA', 'N/A')}")
```

### Finding Paths from an Endpoint

```python
endpoint = list(graph.endpoints)[0]

# Paths starting from this endpoint
for path_id in graph.start_paths.get(endpoint, []):
    record = graph.paths[path_id]
    end_node = record["path"][-1]
    print(f"Path to {end_node}")
```

### Navigating the Hierarchy

```python
from bridge.utils.utils_components import load_all_components_by_layer

components = load_all_components_by_layer("network/hierarchy")

# Top-level component
max_layer = max(components.keys())
top = components[max_layer][0]

# Find subcomponents
for sub_id in top.subcomponents:
    sub = load_component(sub_id, "network/hierarchy")
    print(f"Subcomponent {sub_id}: {len(sub.paths)} paths")
```

## Linked Components Structure

Cross-boundary links are stored as:

```python
graph.linked_components = {
    "other_component_id": {
        "endpoint_in_this": ["endpoint_in_other_1", "endpoint_in_other_2"],
        ...
    },
    ...
}
```

## Storage Format

Components are stored as Zarr-like directories:

```
sub-01_res-0.02mm_lvl-0_cid-0-0-0.zarr/
├── .zattrs          # Component metadata (JSON)
├── paths/           # Path data
│   └── .zarray
├── nodes/           # Node set
│   └── .zarray
└── ...
```

## See Also

- [Node Class](node.md)
- [Core 2: Local Graphs](core2.md)
- [Core 3: Hierarchical Network](core3.md)
