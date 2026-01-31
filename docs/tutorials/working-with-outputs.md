# Working with Outputs

This tutorial teaches you how to explore, visualize, and export BRIDGE outputs after running the pipeline.

## What You'll Learn

- Understanding the output file structure
- Loading and inspecting network components
- Generating summary statistics
- Exporting streamlines for visualization

## Prerequisites

- Completed the [First Tractography](first-tractography.md) tutorial
- A successful BRIDGE run with output files

## Understanding the Output Structure

After running BRIDGE, your output directory contains:

```
derivatives/BRIDGE/sub-test/
├── network/
│   ├── nodes/           # Binary node files per chunk
│   │   └── *.bin
│   ├── hierarchy/       # Graph components at each level
│   │   ├── lvl_0/       # Chunk-level graphs
│   │   ├── lvl_1/       # First merge level
│   │   └── lvl_N/       # Final unified graph
│   └── nifti/           # Reference header information
├── logs/                # Processing logs
└── outputs/             # Generated exports (streamlines, etc.)
```

## Loading the Network

### Load the Final Unified Graph

```python
from bridge.utils.utils_components import (
    load_all_components_by_layer,
    find_max_level
)

# Find and load the highest-level (unified) network
network_dir = "derivatives/BRIDGE/sub-test/network/hierarchy"
max_layer = find_max_level(network_dir)
components = load_all_components_by_layer(network_dir)

# The top level typically has one component
network = components[max_layer][0]

print(f"Network ID: {network.id}")
print(f"Layer: {network.layer}")
print(f"Nodes: {len(network.nodes)}")
print(f"Endpoints: {len(network.endpoints)}")
print(f"Paths: {len(network.paths)}")
```

### Inspect a Specific Layer

```python
# Load all components from a specific layer
layer_0_components = components[0]

print(f"Layer 0 has {len(layer_0_components)} components")
for comp in layer_0_components[:5]:  # First 5
    print(f"  {comp.id}: {len(comp.nodes)} nodes, {len(comp.paths)} paths")
```

## Exploring Paths

Paths are stored as dictionaries with node sequences and quality scores:

```python
# Examine path structure
for path_id, record in list(network.paths.items())[:3]:
    print(f"\nPath: {path_id}")
    print(f"  Nodes in path: {len(record['path'])}")
    print(f"  Length: {record.get('length', 'N/A')}")
    
    scores = record.get('scores', {})
    if scores:
        print(f"  FA: {scores.get('FA', 'N/A'):.3f}")
        print(f"  Alignment: {scores.get('alignment', 'N/A'):.1f}°")
```

### Path Endpoints

```python
# Find which paths connect specific endpoints
endpoint = list(network.endpoints)[0]

# Paths starting from this endpoint
starting_paths = network.start_paths.get(endpoint, [])
print(f"Paths starting at {endpoint}: {len(starting_paths)}")

# Paths ending at this endpoint
ending_paths = network.end_paths.get(endpoint, [])
print(f"Paths ending at {endpoint}: {len(ending_paths)}")
```

## Generating Statistics

### Using the Built-in Overview Tool

```python
from bridge.analysis.graph_overview import BRIDGE_graph_overview

BRIDGE_graph_overview(
    directory="/path/to/project",
    bridge_label="BRIDGE",
    sub_id="test"
)
```

This generates:
- `graph_summary.txt` — Text summary of all layers
- `path_statistics.png` — Distribution plots of path attributes
- `multilayer_summary_graph.tsv` — Network structure for visualization

### Custom Statistics

```python
import numpy as np

# Collect all path scores
fa_values = []
lengths = []
alignments = []

for record in network.paths.values():
    scores = record.get('scores', {})
    if scores.get('FA'):
        fa_values.append(scores['FA'])
    if record.get('length'):
        lengths.append(record['length'])
    if scores.get('alignment'):
        alignments.append(scores['alignment'])

print(f"FA: mean={np.mean(fa_values):.3f}, std={np.std(fa_values):.3f}")
print(f"Length: mean={np.mean(lengths):.1f}, range=[{min(lengths)}, {max(lengths)}]")
print(f"Alignment: mean={np.mean(alignments):.1f}°")
```

## Loading Node Data

Nodes are stored in binary format for efficiency:

```python
from bridge.utils.utils_nodes import load_all_nodes_with_metadata

# Load nodes from a specific chunk
node_file = "derivatives/BRIDGE/sub-test/network/nodes/sub-test_res-0.02mm_cid-0-0-0_nodes.bin"
nodes, metadata = load_all_nodes_with_metadata(node_file)

print(f"Chunk {metadata['chunk_id']}:")
print(f"  Origin: {metadata['origin']}")
print(f"  Dimensions: {metadata['chunk_dims']}")
print(f"  Node count: {len(nodes)}")

# Inspect individual nodes
for node in nodes[:3]:
    print(f"\n  Node {node.id}:")
    print(f"    Centre: {node._centre}")
    print(f"    FA: {node.fa:.3f}")
    print(f"    Principal eigenvector: {node.principal_eigenvector}")
```

## Exporting Streamlines

Convert paths to TrackVis format for visualization in tools like DSI Studio or MI-Brain:

```python
from bridge.analysis.streamlines import BRIDGE_streamlines

BRIDGE_streamlines(
    directory="/path/to/project",
    config_path="streamline_config.json",
    sub_id="test",
    bridge_label="BRIDGE"
)
```

Streamline configuration example:

```json
{
  "sub_id": "test",
  "bridge_label": "BRIDGE",
  "res": 0.02,
  "min_length": 10,
  "max_length": 1000,
  "min_fa": 0.2,
  "batch_size": 10000
}
```

Output appears in:
```
derivatives/BRIDGE/sub-test/outputs/tracts_DD-MM-YY_HHh-MMm/
└── tracts/
    └── batch_0.trk
```

## Visualizing in External Tools

### TrackVis / DSI Studio

1. Open the generated `.trk` file
2. Load your reference NIfTI for anatomical context
3. Apply color coding by FA or other properties

### NetworkX Export

```python
from bridge.analysis.pull_network import BRIDGE_pull_network

# Export to GraphML/GEXF for Gephi, Cytoscape, etc.
G = BRIDGE_pull_network(
    directory="/path/to/project",
    bridge_label="BRIDGE",
    sub_id="test",
    layer=-1,  # Highest layer
    sparse=True
)
```

## Next Steps

- [Export Streamlines in Detail](../how-to/export-outputs.md)
- [Understand Path Scoring](../explanation/pathfinding.md)
- [Configuration Reference](../reference/config-options.md)
