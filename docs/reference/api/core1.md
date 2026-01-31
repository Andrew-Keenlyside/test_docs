# Core 1: Node Extraction

Core 1 extracts candidate fiber nodes from 3D image volumes using structure tensor analysis.

## Main Entry Point

### `extract_nodes`

```python
from bridge.core.core1_nodes import extract_nodes

extract_nodes(
    bridge_path: str,
    omezarr_path: str,
    chunk_size: tuple[int, int, int],
    radius: int,
    sampling_density: float,
    rho: int,
    sigma: int,
    truncate: int,
    mask_min: float,
    mask_max: float,
    min_fa: float,
    sub_id: str,
    res: float,
    overwrite: bool = True
)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `bridge_path` | str | Output directory for BRIDGE results |
| `omezarr_path` | str | Path to NIfTI-Zarr input volume |
| `chunk_size` | tuple | Processing chunk dimensions (z, y, x) |
| `radius` | int | Local z-score window radius |
| `sampling_density` | float | Fraction of mask voxels to sample |
| `rho` | int | Structure tensor outer scale |
| `sigma` | int | Structure tensor inner scale |
| `truncate` | int | Gaussian truncation factor |
| `mask_min` | float | Minimum z-score for mask |
| `mask_max` | float | Maximum z-score for mask |
| `min_fa` | float | Minimum FA threshold |
| `sub_id` | str | Subject identifier |
| `res` | float | Voxel resolution in mm |
| `overwrite` | bool | Whether to overwrite existing outputs |

**Returns:** None (writes to disk)

## Supporting Functions

### `extract_padded_chunk`

Extract a chunk from the volume with boundary padding.

```python
chunk = extract_padded_chunk(
    volume,      # Dask or NumPy array
    origin,      # (z, y, x) start coordinates
    chunk_size,  # (dz, dy, dx) dimensions
    padding      # Padding size in voxels
)
```

### `local_z_score`

Compute local z-score for intensity normalization.

```python
z_scores = local_z_score(volume, radius)
# z = (I - μ_local) / σ_local
```

### `mask_z_score`

Create binary mask from z-score values.

```python
mask = mask_z_score(z_scores, lower=0, upper=2)
```

### `compute_fa_from_vals`

Compute fractional anisotropy from eigenvalues.

```python
fa = compute_fa_from_vals(eigenvalues)
# eigenvalues: shape (3, Z, Y, X)
# fa: shape (Z, Y, X)
```

### `sample_mask`

Randomly sample coordinates from a binary mask.

```python
coords = sample_mask(mask, density=0.01)
# coords: shape (N, 3) float32
```

## ODF-Based Node Extraction

For pre-computed orientation distribution functions:

### `extract_odf_nodes`

```python
from bridge.core.core1_nodes_from_ODF import extract_odf_nodes

extract_odf_nodes(
    bridge_path: str,
    odf_path: str,
    chunk_size: tuple[int, int, int],
    sampling_density: float,
    min_fa: float,
    sub_id: str,
    res: float
)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `odf_path` | str | Path to ODF NIfTI file (SH coefficients) |
| `min_fa` | float | Minimum GFA threshold |

### `trilinear_sample_odf`

Interpolate ODF at continuous coordinates.

```python
odf_vec = trilinear_sample_odf(odf_field, z, y, x)
# odf_vec: shape (n_dirs,)
```

### `gfa_from_odf`

Compute generalized fractional anisotropy from ODF samples.

```python
gfa = gfa_from_odf(odf_samples, axis=-1)
```

## Processing Pipeline

```
Input Volume
    ↓
[extract_padded_chunk] → Chunk with padding
    ↓
[local_z_score] → Normalized intensities
    ↓
[mask_z_score] → Binary mask
    ↓
[structure_tensor.eig] → Eigenvalues, eigenvectors
    ↓
[compute_fa_from_vals] → FA map
    ↓
[sample_mask] → Candidate coordinates
    ↓
[save_chunk_nodes] → Binary node files
```

## Output Format

Node files are written to `{bridge_path}/network/nodes/`:

```
sub-{sub_id}_res-{res}mm_cid-{chunk_id}_nodes.bin
```

See [Node Class](node.md) for binary format details.

## GPU Acceleration

Core 1 uses CuPy for GPU-accelerated operations:

- Chunk extraction and padding
- Local z-score computation
- Structure tensor analysis (via structure-tensor package)
- FA computation
- Mask sampling

Memory usage scales with `chunk_size`. For limited GPU memory, reduce chunk dimensions.

## See Also

- [Node Class](node.md)
- [Configuration Options](../config-options.md)
- [Node Extraction Explained](../../explanation/node-extraction.md)
