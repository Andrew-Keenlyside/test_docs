# File Formats

Reference for all file formats used by BRIDGE.

## Directory Structure

```
project/
├── sub-{sub_id}/
│   └── anat/
│       └── sub-{sub_id}_res-{res}mm_desc-{desc}.niftizarr/
│
└── derivatives/
    └── BRIDGE/
        └── sub-{sub_id}/
            ├── network/
            │   ├── nodes/
            │   │   └── *.bin
            │   ├── hierarchy/
            │   │   ├── lvl_0/
            │   │   │   └── *.zarr/
            │   │   ├── lvl_1/
            │   │   └── lvl_N/
            │   └── nifti/
            │       └── .zattrs
            ├── logs/
            │   ├── setup_log.txt
            │   ├── nodes_log.txt
            │   ├── local_graph_log.txt
            │   └── hierarchy_log.txt
            └── outputs/
                └── tracts_{timestamp}/
                    └── tracts/
                        └── *.trk
```

## Input Formats

### NIfTI (.nii, .nii.gz)

Standard neuroimaging format. BRIDGE converts to NIfTI-Zarr internally.

### NIfTI-Zarr (.niftizarr)

Chunked, cloud-optimized format based on OME-NGFF:

```
volume.niftizarr/
├── .zattrs           # OME-NGFF metadata
├── .zgroup
├── 0/                # Full resolution
│   ├── .zarray
│   └── 0.0.0, 0.0.1, ...  # Chunks
├── 1/                # 2x downsampled
├── 2/                # 4x downsampled
└── nifti/            # NIfTI header
    ├── .zattrs       # Header as JSON
    └── .zarray
```

## Node Files (.bin)

Binary format for efficient node storage.

### File Structure

```
Offset  Size    Content
------  ------  -------
0       4       Magic number: b'NOD3'
4       4       Header size (uint32, little-endian)
8       N       Header JSON (N bytes)
8+N     M×37    Node records (M nodes, 37 bytes each)
```

### Header JSON

```json
{
  "record_size": 37,
  "node_count": 15234,
  "metadata": {
    "chunk_id": "0-0-0",
    "origin": [0, 0, 0],
    "chunk_dims": [256, 256, 256]
  }
}
```

### Node Record Format

```python
NODE_STRUCT_FMT = '<i3f3fBff??'

# Field         Offset  Size  Type      Description
# -----         ------  ----  ----      -----------
# id            0       4     int32     Node identifier
# centre[0]     4       4     float32   Z coordinate
# centre[1]     8       4     float32   Y coordinate
# centre[2]     12      4     float32   X coordinate
# pe[0]         16      4     float32   Eigenvector Z
# pe[1]         20      4     float32   Eigenvector Y
# pe[2]         24      4     float32   Eigenvector X
# img           28      1     uint8     Intensity value
# fa            29      4     float32   Fractional anisotropy
# local_z       33      4     float32   Local z-score
# is_endpoint   37      1     bool      Endpoint flag
# searched      38      1     bool      Searched flag
# Total: 37 bytes
```

## Graph Components (.zarr)

Zarr-based storage for graph components.

### Directory Structure

```
sub-01_res-0.02mm_lvl-0_cid-0-0-0.zarr/
├── .zattrs           # Component metadata
├── .zgroup
├── paths/
│   ├── .zarray
│   └── 0             # Path data (JSON)
├── nodes/
│   ├── .zarray
│   └── 0             # Node set
├── endpoints/
│   ├── .zarray
│   └── 0             # Endpoint set
└── linked_components/
    ├── .zarray
    └── 0             # Link data
```

### Component .zattrs

```json
{
  "id": "sub-01_res-0.02mm_lvl-0_cid-0-0-0",
  "layer": 0,
  "lowest_layer": true,
  "chunk_ids": ["0-0-0"],
  "centre_coords": [128.5, 128.5, 128.5],
  "classified_nodes": false,
  "subcomponents": []
}
```

### Path Record

```json
{
  "path": ["node_id_1", "node_id_2", "node_id_3"],
  "length": 3,
  "scores": {
    "FA": 0.45,
    "intensity": 180,
    "local_z": 1.2,
    "alignment": 15.3,
    "path_distance": 42.5,
    "edge_distance": 14.2,
    "bending_angle": 8.5,
    "curve_radius": 120.0
  }
}
```

## Streamline Files (.trk)

TrackVis format for fiber visualization.

### Header (1000 bytes)

| Field | Offset | Size | Type | Description |
|-------|--------|------|------|-------------|
| id_string | 0 | 6 | char | "TRACK\0" |
| dim | 6 | 6 | int16[3] | Volume dimensions |
| voxel_size | 12 | 12 | float32[3] | Voxel size in mm |
| origin | 24 | 12 | float32[3] | Origin coordinates |
| n_scalars | 36 | 2 | int16 | Scalars per point |
| scalar_name | 38 | 200 | char[10][20] | Scalar names |
| n_properties | 238 | 2 | int16 | Properties per tract |
| property_name | 240 | 200 | char[10][20] | Property names |
| vox_to_ras | 440 | 64 | float32[4][4] | Affine matrix |
| ... | | | | |
| n_count | 988 | 4 | int32 | Number of tracks |
| version | 992 | 4 | int32 | Version (2) |
| hdr_size | 996 | 4 | int32 | Header size (1000) |

### Track Data

```
For each track:
  n_points (int32)
  For each point:
    x, y, z (float32[3])
    scalars (float32[n_scalars])  # if any
  properties (float32[n_properties])  # if any
```

## Log Files

Plain text logs with timestamps:

```
[2025-01-15T10:30:45.123456] Starting extract nodes
----- BEGIN CHUNK 0-0-0 -----
[2025-01-15T10:30:45.234567] [0-0-0] [INFO] Starting
[2025-01-15T10:30:46.345678] [0-0-0] [INFO] GPU mem Free=20.5 GB
[2025-01-15T10:30:50.456789] [0-0-0] [INFO] 15234 nodes saved
----- END CHUNK 0-0-0 -----
```

## BIDS Naming Convention

Files follow BIDS-like naming:

```
sub-{subject}_res-{resolution}mm_desc-{description}_cid-{chunk_id}_suffix.ext
```

Examples:
- `sub-01_res-0.02mm_cid-0-0-0_nodes.bin`
- `sub-01_res-0.02mm_lvl-0_cid-0-0-0.zarr`
- `sub-01_res-0.02mm_desc-brain.niftizarr`

## See Also

- [Node Class](api/node.md)
- [Graph Class](api/graph.md)
- [BIDS Specification](https://bids-specification.readthedocs.io/)
