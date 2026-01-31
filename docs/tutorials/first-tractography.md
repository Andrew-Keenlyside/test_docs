# Your First Tractography

This tutorial walks you through running the complete BRIDGE pipeline on sample data. By the end, you'll have generated your first white matter tractography from 3D microscopy images.

## What You'll Learn

- How to structure your input data
- How to configure the pipeline
- How to run each processing stage
- How to verify your results

## Prerequisites

- BRIDGE installed and working
- Sample data downloaded (see [Tutorials index](index.md))
- ~10GB free disk space
- GPU with at least 8GB VRAM

## Step 1: Organize Your Data

BRIDGE expects a BIDS-like directory structure:

```
project/
├── sub-test/
│   └── anat/
│       └── sub-test_res-0.02mm_desc-sample.nii.gz
└── derivatives/
    └── BRIDGE/
        └── sub-test/
            ├── network/
            └── logs/
```

Create this structure for your sample data:

```python
import os
from pathlib import Path

project_dir = Path("/path/to/your/project")
sub_id = "test"

# Create directories
(project_dir / f"sub-{sub_id}" / "anat").mkdir(parents=True, exist_ok=True)
(project_dir / "derivatives" / "BRIDGE" / f"sub-{sub_id}").mkdir(parents=True, exist_ok=True)

# Copy or symlink your input image
# cp sample_brain.nii.gz project/sub-test/anat/sub-test_res-0.02mm_desc-sample.nii.gz
```

## Step 2: Create a Configuration File

Create a JSON configuration file that defines processing parameters:

```json
{
  "sub_id": "test",
  "desc": "sample",
  "resolution": [0.02, 0.02, 0.02],
  "threads": 8,
  "overwrite": true,

  "run_core1": true,
  "core1": {
    "chunk_size": [256, 256, 256],
    "radius": 32,
    "sampling_density": 0.005,
    "rho": 3,
    "sigma": 1,
    "truncate": 2,
    "min_fa": 0.3
  },

  "run_core2": true,
  "core2": {
    "max_edge_distance": 20,
    "edge_allignment_limit": 25,
    "segment_radius": 60
  },

  "run_core3": true
}
```

Save this as `config.json` in your project directory.

### Understanding the Key Parameters

| Parameter | What it controls |
|-----------|------------------|
| `resolution` | Voxel size in mm (must match your image) |
| `chunk_size` | Processing tile size — larger uses more GPU memory |
| `sampling_density` | Fraction of valid voxels to sample as nodes |
| `min_fa` | Minimum fractional anisotropy to include a voxel |
| `max_edge_distance` | Maximum distance (µm) for connecting nodes |

## Step 3: Run the Pipeline

### Option A: Run All Stages Together

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    input_dir="/path/to/project/sub-test/anat",
    project_dir="/path/to/project",
    config="config.json"
)
```

### Option B: Run Stages Individually

For more control, run each stage separately:

```python
from bridge import BRIDGE_setup

# Stage 1: Extract nodes from image
BRIDGE_setup(
    input_dir="/path/to/project/sub-test/anat",
    project_dir="/path/to/project",
    config="config.json",
    run_core1=True,
    run_core2=False,
    run_core3=False
)
```

```python
# Stage 2: Build local graphs
BRIDGE_setup(
    input_dir=None,  # Not needed after Stage 1
    project_dir="/path/to/project",
    config="config.json",
    run_core1=False,
    run_core2=True,
    run_core3=False
)
```

```python
# Stage 3: Hierarchical merging
BRIDGE_setup(
    input_dir=None,
    project_dir="/path/to/project",
    config="config.json",
    run_core1=False,
    run_core2=False,
    run_core3=True
)
```

## Step 4: Monitor Progress

Each stage writes logs to `derivatives/BRIDGE/sub-test/logs/`:

```bash
# Watch node extraction progress
tail -f derivatives/BRIDGE/sub-test/logs/nodes_log.txt

# Watch local graph construction
tail -f derivatives/BRIDGE/sub-test/logs/local_graph_log.txt

# Watch hierarchical merging
tail -f derivatives/BRIDGE/sub-test/logs/hierarchy_log.txt
```

## Step 5: Verify Your Results

After the pipeline completes, check the output structure:

```
derivatives/BRIDGE/sub-test/
├── network/
│   ├── nodes/
│   │   ├── sub-test_res-0.02mm_cid-0-0-0_nodes.bin
│   │   ├── sub-test_res-0.02mm_cid-0-0-1_nodes.bin
│   │   └── ...
│   └── hierarchy/
│       ├── lvl_0/
│       │   └── ... (chunk-level graphs)
│       ├── lvl_1/
│       │   └── ... (merged graphs)
│       └── lvl_N/
│           └── ... (final unified graph)
└── logs/
    ├── setup_log.txt
    ├── nodes_log.txt
    ├── local_graph_log.txt
    └── hierarchy_log.txt
```

### Quick Verification

```python
from bridge.utils.utils_components import load_all_components_by_layer, find_max_level

network_dir = "derivatives/BRIDGE/sub-test/network/hierarchy"
max_layer = find_max_level(network_dir)

components = load_all_components_by_layer(network_dir)
top_level = components[max_layer][0]

print(f"Final network has:")
print(f"  - {len(top_level.nodes)} nodes")
print(f"  - {len(top_level.paths)} paths")
print(f"  - {len(top_level.endpoints)} endpoints")
```

## Troubleshooting

### "CUDA out of memory"

Reduce `chunk_size` in your config:
```json
"chunk_size": [128, 128, 128]
```

### "No nodes found in chunk"

Your `min_fa` threshold may be too high for your data. Try lowering it:
```json
"min_fa": 0.2
```

### Pipeline hangs on Core 2

Check GPU memory with `nvidia-smi`. Core 2 builds spatial indices that can be memory-intensive. Reduce `max_edge_distance` if needed.

## Next Steps

Now that you've run the pipeline:

1. **Explore your results** — Continue to [Working with Outputs](working-with-outputs.md)
2. **Extract streamlines** — See [How to Export Streamlines](../how-to/export-outputs.md)
3. **Understand the method** — Read [Graph Construction Explained](../explanation/graph-construction.md)
