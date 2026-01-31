# Run the Full Pipeline

This guide covers running the complete BRIDGE pipeline from start to finish.

## Prerequisites

- BRIDGE installed ([Installation Guide](install.md))
- Configuration file ready ([Configuration Guide](configure.md))
- Input data organized in BIDS-like structure

## Quick Start

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    input_dir="/data/project/sub-01/anat",
    project_dir="/data/project",
    config="config.json"
)
```

This runs all three stages sequentially:
1. **Core 1**: Node extraction (GPU)
2. **Core 2**: Local graph construction (GPU)
3. **Core 3**: Hierarchical merging (CPU)

## Running Individual Stages

### Stage 1: Node Extraction

Extract candidate fiber nodes from the image volume:

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    input_dir="/data/project/sub-01/anat",
    project_dir="/data/project",
    config="config.json",
    run_core1=True,
    run_core2=False,
    run_core3=False
)
```

**What it does:**
- Converts input to NIfTI-Zarr format (if needed)
- Processes volume in chunks
- Computes structure tensor at each voxel
- Samples nodes based on FA threshold
- Saves binary node files per chunk

**Output:** `network/nodes/*.bin`

### Stage 2: Local Graph Construction

Build graphs within each chunk:

```python
BRIDGE_setup(
    input_dir=None,  # Not needed
    project_dir="/data/project",
    config="config.json",
    run_core1=False,
    run_core2=True,
    run_core3=False
)
```

**What it does:**
- Loads nodes with overlap between chunks
- Builds spatial KD-tree for neighbor lookup
- Creates edges based on distance and alignment
- Defines segments (node triplets)
- Runs minimum spanning forest
- Extracts paths through BFS

**Output:** `network/hierarchy/lvl_0/*.zarr`

### Stage 3: Hierarchical Merging

Merge chunk-level graphs into unified network:

```python
BRIDGE_setup(
    input_dir=None,
    project_dir="/data/project",
    config="config.json",
    run_core1=False,
    run_core2=False,
    run_core3=True
)
```

**What it does:**
- Groups adjacent chunk components
- Finds cross-boundary endpoint links
- Merges connected components
- Repeats hierarchically until single network
- Simplifies final representation

**Output:** `network/hierarchy/lvl_N/*.zarr` (where N = max level)

## Monitoring Progress

### Log Files

Each stage writes detailed logs:

```bash
# Real-time monitoring
tail -f derivatives/BRIDGE/sub-01/logs/nodes_log.txt
tail -f derivatives/BRIDGE/sub-01/logs/local_graph_log.txt
tail -f derivatives/BRIDGE/sub-01/logs/hierarchy_log.txt
```

### GPU Monitoring

Watch GPU utilization during Core 1 and 2:

```bash
watch -n 1 nvidia-smi
```

### Dask Dashboard

When using Dask distributed, access the dashboard:

```python
# Dashboard URL printed when pipeline starts
# Usually: http://localhost:8787
```

## Handling Large Volumes

### Memory Management

For volumes larger than GPU memory:

```json
{
  "core1": {
    "chunk_size": [128, 128, 128]
  },
  "threads": 2,
  "GPU_limit": 0.8
}
```

### Resuming Interrupted Runs

BRIDGE can resume from where it stopped:

```python
BRIDGE_setup(
    input_dir=None,
    project_dir="/data/project",
    config="config.json",
    overwrite=False  # Skip completed chunks
)
```

### Processing Specific Chunks

<!-- Placeholder: Add chunk-specific processing when implemented -->
```python
# Future feature: process specific chunks only
# BRIDGE_setup(..., chunks=["0-0-0", "0-0-1", "0-1-0"])
```

## Common Workflows

### Full Pipeline with Default Config

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    input_dir="/data/sub-01/anat",
    project_dir="/data",
    sub_id="01",
    desc="brain",
    resolution=[0.02, 0.02, 0.02]
)
```

### Re-run Graph Construction Only

After adjusting Core 2 parameters:

```python
BRIDGE_setup(
    input_dir=None,
    project_dir="/data",
    config="config.json",
    run_core1=False,
    run_core2=True,
    run_core3=True,
    overwrite=True  # Regenerate graphs
)
```

### Process Multiple Subjects

```python
subjects = ["01", "02", "03"]

for sub in subjects:
    BRIDGE_setup(
        input_dir=f"/data/sub-{sub}/anat",
        project_dir="/data",
        config="config.json",
        sub_id=sub
    )
```

## Troubleshooting

### Core 1 Issues

**"Empty chunk" warnings:**
- Normal for chunks outside brain tissue
- Check `min_fa` if too many chunks are empty

**CUDA out of memory:**
- Reduce `chunk_size`
- Lower `sampling_density`

### Core 2 Issues

**"No edges after refinement":**
- Increase `max_edge_distance`
- Lower `edge_allignment_limit`

**Very slow processing:**
- Reduce `max_edge_distance` (fewer edges to process)
- Check GPU utilization with `nvidia-smi`

### Core 3 Issues

**"No components to merge":**
- Verify Core 2 completed successfully
- Check `network/hierarchy/lvl_0/` for outputs

## Next Steps

- [Export Streamlines](export-outputs.md)
- [Run on HPC Clusters](run-on-cluster.md)
- [Understand the Pipeline Architecture](../explanation/architecture.md)
