# BRIDGE

<img src="assets/BRIDGE_logo_light.png#only-light" alt="UCL-BRIDGE" width="60%" />
<img src="assets/BRIDGE_logo_dark.png#only-dark"  alt="UCL-BRIDGE" width="60%" />


**B**rain **R**econstruction **I**ntegrating **D**istributed **G**raph **E**stimates

BRIDGE is a GPU-accelerated pipeline for reconstructing white matter fiber pathways from high-resolution 3D microscopy images. It builds a hierarchical graph representation of neural connectivity by:

1. **Extracting nodes** from image data using structure tensor analysis or ODF peak detection
2. **Building local graphs** within spatial chunks using geometric constraints
3. **Merging hierarchically** to produce a unified brain-wide network

## Key Features

- **Multi-scale processing** — Handles terabyte-scale volumes through chunked processing
- **GPU acceleration** — Core stages use RAPIDS (cuDF, cuGraph, CuPy) for fast computation
- **Parameter-light design** — Geometric constraints replace manual threshold tuning
- **BIDS-compatible outputs** — Standardized file naming and directory structure
- **Streamline export** — Generate TrackVis (.trk) files for visualization

## Quick Start

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    input_dir="/path/to/image/stack",
    project_dir="/path/to/output",
    config="config.json"
)
```

## Documentation Structure

This documentation follows the [Diátaxis framework](https://diataxis.fr/):

| Section | Purpose |
|---------|---------|
| [**Tutorials**](tutorials/index.md) | Learning-oriented guides for newcomers |
| [**How-to Guides**](how-to/index.md) | Task-oriented recipes for specific goals |
| [**Reference**](reference/index.md) | Technical specifications and API details |
| [**Explanation**](explanation/index.md) | Conceptual background and design rationale |

## Requirements

- Python 3.10+
- CUDA-capable GPU (compute capability 7.0+)
- RAPIDS libraries (cuDF, cuGraph, CuPy)
- 32GB+ RAM recommended for large volumes

## Citation

If you use BRIDGE in your research, please cite:

```bibtex
@article{keenlyside2025bridge,
  title={BRIDGE: Brain Reconstruction via Iterative Directed Graph Extraction},
  author={Keenlyside, Andrew},
  year={2025}
}
```

## License

BRIDGE is released under the MIT License.
