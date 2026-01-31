# Export Streamlines

This guide explains how to export BRIDGE paths as TrackVis streamlines for visualization.

## Basic Export

```python
from bridge.analysis.streamlines import BRIDGE_streamlines

BRIDGE_streamlines(
    directory="/data/project",
    sub_id="01",
    bridge_label="BRIDGE",
    res=0.02
)
```

Output: `derivatives/BRIDGE/sub-01/outputs/tracts_DD-MM-YY_HHh-MMm/tracts/*.trk`

## Using a Configuration File

Create a streamline configuration:

```json
{
  "sub_id": "01",
  "bridge_label": "BRIDGE",
  "res": 0.02,
  "desc": "brain",
  
  "min_length": 10,
  "max_length": 5000,
  "min_fa": 0.2,
  "min_path_nodes": 5,
  
  "batch_size": 10000,
  "max_size_gb": 2.0,
  
  "tract_smoothing": false,
  "tract_binning": false,
  "per_region": false
}
```

```python
BRIDGE_streamlines(
    directory="/data/project",
    config_path="streamline_config.json"
)
```

## Filtering Paths

### By Path Properties

```python
BRIDGE_streamlines(
    directory="/data/project",
    sub_id="01",
    bridge_label="BRIDGE",
    res=0.02,
    
    # Length filters
    min_length=20,
    max_length=1000,
    
    # Quality filters
    min_fa=0.25,
    max_fa=0.9,
    
    # Node count
    min_path_nodes=10
)
```

### By Region Masks

Filter paths that start or end in specific regions:

```python
import numpy as np

# Define origin mask (voxel coordinates)
origin_coords = np.array([
    [100, 150, 200],
    [101, 150, 200],
    # ... more coordinates
])

# Define terminus mask
terminus_coords = np.array([
    [300, 200, 150],
    [301, 200, 150],
])

BRIDGE_streamlines(
    directory="/data/project",
    sub_id="01",
    bridge_label="BRIDGE",
    origin_mask=origin_coords,
    terminus_mask=terminus_coords
)
```

## Per-Region Export

Export separate tract files for each anatomical region:

```python
BRIDGE_streamlines(
    directory="/data/project",
    sub_id="01",
    bridge_label="BRIDGE",
    per_region=True,
    LUT="freesurfer",
    freesurfer_path="/path/to/freesurfer"
)
```

Output structure:
```
tracts/
├── Left-Hippocampus.trk
├── Right-Hippocampus.trk
├── Left-Thalamus.trk
└── ...
```

## Batch Processing

For large networks, paths are exported in batches:

```python
BRIDGE_streamlines(
    directory="/data/project",
    sub_id="01",
    bridge_label="BRIDGE",
    batch_size=50000,      # Paths per file
    max_size_gb=1.0        # Max file size
)
```

Output:
```
tracts/
├── batch_0.trk
├── batch_1.trk
└── batch_2.trk
```

## Output Format

### TrackVis (.trk) Files

BRIDGE exports standard TrackVis format compatible with:
- DSI Studio
- MI-Brain
- TrackVis viewer
- Nibabel

### Included Properties

Each streamline includes per-point or per-tract properties:

| Property | Type | Description |
|----------|------|-------------|
| `FA` | per-tract | Mean fractional anisotropy |
| `length` | per-tract | Path length in nodes |
| `alignment` | per-tract | Mean edge-vector alignment |
| `intensity` | per-tract | Mean image intensity |

## Loading Exported Streamlines

### With Nibabel

```python
import nibabel as nib

trk = nib.streamlines.load("tracts/batch_0.trk")
streamlines = trk.streamlines

print(f"Number of streamlines: {len(streamlines)}")
print(f"First streamline shape: {streamlines[0].shape}")

# Access per-tract data
data = trk.tractogram.data_per_streamline
print(f"Available properties: {list(data.keys())}")
```

### With DIPY

```python
from dipy.io.streamline import load_trk

tractogram = load_trk("tracts/batch_0.trk", reference="same")
streamlines = tractogram.streamlines

# Filter by property
fa_values = tractogram.data_per_streamline['FA']
high_fa = streamlines[fa_values > 0.3]
```

## Visualization

### DSI Studio

1. Open DSI Studio
2. File → Open → Select .trk file
3. Load reference NIfTI for anatomical context
4. Adjust rendering options

### Python (Matplotlib)

```python
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D
import nibabel as nib

trk = nib.streamlines.load("tracts/batch_0.trk")

fig = plt.figure(figsize=(10, 10))
ax = fig.add_subplot(111, projection='3d')

for sl in list(trk.streamlines)[:1000]:  # First 1000
    ax.plot(sl[:, 0], sl[:, 1], sl[:, 2], 
            alpha=0.3, linewidth=0.5)

ax.set_xlabel('X')
ax.set_ylabel('Y')
ax.set_zlabel('Z')
plt.show()
```

## Troubleshooting

### Empty Output

**No streamlines generated:**
- Check that network has paths: `len(network.paths) > 0`
- Lower filter thresholds
- Verify node coordinates are correct

### Coordinate Mismatch

**Streamlines don't align with anatomy:**
- Check `resolution` matches your image
- Verify NIfTI header in `network/nifti/.zattrs`

### Memory Issues

**Out of memory during export:**
- Reduce `batch_size`
- Lower `max_size_gb`

## Next Steps

- [Extract Image Chunks](extract-chunks.md)
- [Export to NetworkX](export-networkx.md)
- [Visualization in External Tools](../tutorials/working-with-outputs.md#visualizing-in-external-tools)
