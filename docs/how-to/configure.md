# Configuration Files

This guide explains how to create and customize BRIDGE configuration files.

## Basic Configuration

Create a JSON file with your processing parameters:

```json
{
  "sub_id": "01",
  "desc": "brain",
  "resolution": [0.02, 0.02, 0.02],
  "threads": 8,
  "overwrite": true,

  "run_core1": true,
  "run_core2": true,
  "run_core3": true,

  "core1": {
    "chunk_size": [256, 256, 256],
    "radius": 32,
    "sampling_density": 0.005,
    "rho": 3,
    "sigma": 1,
    "truncate": 2,
    "min_fa": 0.3
  },

  "core2": {
    "max_edge_distance": 20,
    "edge_allignment_limit": 25,
    "segment_radius": 60,
    "min_component_size": 5
  }
}
```

## Using Configuration Files

### Pass to BRIDGE_setup

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    input_dir="/path/to/images",
    project_dir="/path/to/project",
    config="config.json"
)
```

### Override Specific Parameters

```python
BRIDGE_setup(
    input_dir="/path/to/images",
    project_dir="/path/to/project",
    config="config.json",
    # Override config file values:
    sampling_density=0.01,
    min_fa=0.25
)
```

## Parameter Reference

### Global Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `sub_id` | string | "default" | Subject identifier for BIDS naming |
| `desc` | string | "default" | Description label for output files |
| `resolution` | [float, float, float] | [1.0, 1.0, 1.0] | Voxel size in mm [z, y, x] |
| `threads` | int | 4 | Worker threads per GPU |
| `overwrite` | bool | true | Overwrite existing outputs |
| `GPU_limit` | float | 0.9 | Maximum GPU memory fraction |

### Core 1: Node Extraction

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `chunk_size` | [int, int, int] | [512, 512, 512] | Processing chunk dimensions |
| `radius` | int | 64 | Local z-score window radius |
| `sampling_density` | float | 0.01 | Fraction of voxels to sample |
| `rho` | int | 5 | Structure tensor outer scale |
| `sigma` | int | 3 | Structure tensor inner scale |
| `truncate` | int | 3 | Gaussian truncation factor |
| `min_fa` | float | 0.3 | Minimum FA threshold |
| `mask_min` | float | 0 | Minimum z-score for masking |
| `mask_max` | float | 2 | Maximum z-score for masking |

### Core 2: Local Graph Construction

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_edge_distance` | float | 60 | Maximum node connection distance (µm) |
| `edge_allignment_limit` | float | 60 | Maximum edge-vector angle (degrees) |
| `segment_radius` | float | 45 | Segment grouping radius |
| `segment_max_angle` | float | 45 | Maximum segment angle deviation |
| `min_component_size` | int | 5 | Minimum nodes per component |

## Configuration Presets

### High-Resolution Microscopy (20µm voxels)

```json
{
  "resolution": [0.02, 0.02, 0.02],
  "core1": {
    "chunk_size": [256, 256, 256],
    "sampling_density": 0.005,
    "rho": 3,
    "sigma": 1
  },
  "core2": {
    "max_edge_distance": 20,
    "segment_radius": 60
  }
}
```

### Standard dMRI (1mm voxels)

```json
{
  "resolution": [1.0, 1.0, 1.0],
  "core1": {
    "chunk_size": [64, 64, 64],
    "sampling_density": 0.1,
    "rho": 2,
    "sigma": 1
  },
  "core2": {
    "max_edge_distance": 3,
    "segment_radius": 5
  }
}
```

### Memory-Constrained (8GB GPU)

```json
{
  "core1": {
    "chunk_size": [128, 128, 128],
    "sampling_density": 0.002
  },
  "threads": 2,
  "GPU_limit": 0.7
}
```

## Validating Configuration

Check your config before running:

```python
import json

with open("config.json") as f:
    cfg = json.load(f)

# Required fields
required = ["sub_id", "resolution", "core1", "core2"]
for field in required:
    if field not in cfg:
        print(f"Missing required field: {field}")

# Validate resolution matches your data
print(f"Expected voxel size: {cfg['resolution']} mm")

# Check memory requirements
chunk = cfg["core1"]["chunk_size"]
voxels = chunk[0] * chunk[1] * chunk[2]
mem_gb = voxels * 4 / 1e9  # float32
print(f"Approximate chunk memory: {mem_gb:.1f} GB")
```

## Next Steps

- [Configure GPU Settings](configure-gpu.md)
- [Run the Pipeline](run-pipeline.md)
- [Configuration Options Reference](../reference/config-options.md)
