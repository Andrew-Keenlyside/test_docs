# Configuration Options

Complete reference for all BRIDGE configuration parameters.

## Global Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `sub_id` | string | `"default"` | Subject identifier for BIDS naming |
| `desc` | string | `"default"` | Description label for output files |
| `resolution` | array[3] | `[1.0, 1.0, 1.0]` | Voxel size in mm [z, y, x] |
| `omezarr_levels` | int | `-1` | Pyramid levels to generate (-1 = auto) |
| `overwrite` | bool | `true` | Overwrite existing outputs |
| `GPU_limit` | float | `0.9` | Maximum GPU memory fraction (0.0-1.0) |
| `threads` | int | `4` | Worker threads per GPU |
| `rotation` | array[3] | `[0.0, 0.0, 0.0]` | Volume rotation in degrees [rx, ry, rz] |

## Stage Control

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `run_core1` | bool | `true` | Execute node extraction |
| `run_core2` | bool | `true` | Execute local graph construction |
| `run_core3` | bool | `true` | Execute hierarchical merging |

## Core 1: Node Extraction

| Parameter | Type | Default | Range | Description |
|-----------|------|---------|-------|-------------|
| `chunk_size` | array[3] | `[512, 512, 512]` | 64-1024 | Processing chunk dimensions |
| `radius` | int | `64` | 8-128 | Local z-score window radius |
| `sampling_density` | float | `0.01` | 0.0001-1.0 | Fraction of mask voxels to sample |
| `rho` | int | `5` | 1-10 | Structure tensor outer scale (σ_outer) |
| `sigma` | int | `3` | 1-10 | Structure tensor inner scale (σ_inner) |
| `truncate` | int | `3` | 2-5 | Gaussian truncation factor |
| `mask_min` | float | `0` | -∞ to +∞ | Minimum z-score for inclusion |
| `mask_max` | float | `2` | -∞ to +∞ | Maximum z-score for inclusion |
| `min_fa` | float | `0.3` | 0.0-1.0 | Minimum FA threshold |

### Structure Tensor Parameters

The structure tensor is computed as:

```
S = G_ρ * (∇I ⊗ ∇I)
```

Where:
- `sigma` (σ) controls the gradient computation scale
- `rho` (ρ) controls the integration/smoothing scale
- `truncate` determines how many σ to include in Gaussian kernel

**Guidelines:**
- For 20µm voxels: `rho=3`, `sigma=1`
- For 1mm voxels: `rho=2`, `sigma=1`
- Larger values smooth over more voxels

## Core 2: Local Graph Construction

| Parameter | Type | Default | Range | Description |
|-----------|------|---------|-------|-------------|
| `max_edge_distance` | float | `60` | 1-200 | Maximum node connection distance (µm) |
| `edge_allignment_limit` | float | `60` | 0-90 | Maximum edge-vector angle (degrees) |
| `segment_radius` | float | `45` | 10-100 | Segment grouping radius (µm) |
| `segment_max_angle` | float | `45` | 0-90 | Maximum segment angle deviation |
| `min_component_size` | int | `5` | 1-100 | Minimum nodes per component |
| `max_bad_steps` | int | `1` | 0-5 | Allowed consecutive bad steps in path |
| `max_tortuosity` | float | `1.2` | 1.0-2.0 | Maximum path tortuosity ratio |

### Edge Filtering

Edges are kept if:
1. Distance ≤ `max_edge_distance`
2. Angle between edge and both node vectors ≤ `edge_allignment_limit`
3. No shorter two-hop path exists (triangle shortcut removal)

### Segment Definition

A segment is a node triplet (a, b, c) where:
- Edge a-b exists
- Edge b-c exists
- Total span ≤ `segment_radius`
- Turn angle ≤ `segment_max_angle`

## Streamline Export

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `min_length` | int | `10` | Minimum path length (nodes) |
| `max_length` | int | `5000` | Maximum path length (nodes) |
| `min_path_nodes` | int | `5` | Minimum nodes per exported path |
| `min_fa` | float | `0.2` | Minimum mean FA for export |
| `max_fa` | float | `1.0` | Maximum mean FA for export |
| `min_intensity` | float | `null` | Minimum mean intensity |
| `max_intensity` | float | `null` | Maximum mean intensity |
| `min_local_z` | float | `null` | Minimum mean local z-score |
| `max_local_z` | float | `null` | Maximum mean local z-score |
| `batch_size` | int | `10000` | Paths per output file |
| `max_size_gb` | float | `2.0` | Maximum file size in GB |
| `tract_smoothing` | bool | `false` | Apply Gaussian smoothing |
| `tract_binning` | bool | `false` | Subsample path points |
| `per_region` | bool | `false` | Separate files per region |
| `LUT` | string | `null` | Label lookup table ("freesurfer") |

## Example Configurations

### High-Resolution Microscopy

```json
{
  "sub_id": "mouse01",
  "resolution": [0.02, 0.02, 0.02],
  "core1": {
    "chunk_size": [256, 256, 256],
    "sampling_density": 0.005,
    "rho": 3,
    "sigma": 1,
    "min_fa": 0.3
  },
  "core2": {
    "max_edge_distance": 20,
    "edge_allignment_limit": 30,
    "segment_radius": 60
  }
}
```

### Memory-Constrained

```json
{
  "GPU_limit": 0.7,
  "threads": 2,
  "core1": {
    "chunk_size": [128, 128, 128],
    "sampling_density": 0.002
  }
}
```

### Aggressive Filtering

```json
{
  "core1": {
    "min_fa": 0.4,
    "mask_min": 0.5,
    "mask_max": 1.5
  },
  "core2": {
    "edge_allignment_limit": 20,
    "min_component_size": 10
  }
}
```

## See Also

- [Configuration Guide](../how-to/configure.md)
- [Default Values](defaults.md)
