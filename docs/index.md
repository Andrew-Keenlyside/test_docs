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

## Implementations

BRIDGE is available in two language implementations:

| | BRIDGE.py | BRIDGE.jl |
|--|-----------|-----------|
| **Best for** | GPU stages (Core 1 & 2) | CPU stages (Core 3, streamlines) |
| **Acceleration** | RAPIDS (CuPy, cuGraph) | SuiteSparseGraphBLAS, native threading |
| **Status** | Complete | Core 3 & streamlines complete |

Both implementations share identical algorithms, data formats, and configuration—use them together in [hybrid workflows](how-to/hybrid-workflow.md) for best performance.

## Documentation Structure

| Section | Purpose |
|---------|---------|
| [**Tutorials**](tutorials/index.md) | Learning-oriented guides for newcomers |
| [**How-to Guides**](how-to/index.md) | Task-oriented recipes for specific goals |
| [**Reference**](reference/index.md) | Technical specifications and API details |
| [**Explanation**](explanation/index.md) | Conceptual background and design rationale |


## Quick Start

### Python (GPU)

```bash
pip install bridge-tractography

# Or with RAPIDS
conda install -c rapidsai -c conda-forge cupy cudf cugraph
```

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    "/path/to/input",
    "/path/to/project",
    sub_id="01",
    resolution=(1.0, 1.0, 1.0),
)
```

### Julia (CPU acceleration)

```julia
using Pkg
Pkg.add("BRIDGE")
```

```julia
using BRIDGE

run_BRIDGE_setup_jl(
    "/path/to/input",
    "/path/to/project",
    sub_id="01",
    run_core1=true, 
    run_core2=true,   
    run_core3=true,   # Julia accelerated - built for clusters
)
```

## Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Input: 3D Volume                         │
│                    (NIfTI / NIfTI-Zarr / TIFF)                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  CORE 1: Node Extraction                     [Python GPU best]  │
│  Structure tensor → FA masking → Stochastic sampling            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  CORE 2: Local Graph Construction            [Python GPU best]  │
│  KD-tree edges → Alignment filter → Segments → MSF → Paths      │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  CORE 3: Hierarchical Network                  [Julia CPU best] │
│  Octree grouping → Boundary linking → Path merging              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Output: Streamlines                          │
│                      (TrackVis .trk)                            │
└─────────────────────────────────────────────────────────────────┘
```

## Features

- **Scale**: Process terabyte-scale volumes via chunked processing
- **Speed**: GPU acceleration for compute-intensive stages
- **Quality**: Parameter-free MSF pruning preserves biological topology
- **Interoperability**: BIDS-compliant outputs, TrackVis export
- **Flexibility**: Mix Python and Julia for optimal performance

## Links

- [GitHub Repository](https://github.com/your-org/BRIDGE)
- [Issue Tracker](https://github.com/your-org/BRIDGE/issues)
- [BRIDGE.jl Repository](https://github.com/your-org/BRIDGE.jl)


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
