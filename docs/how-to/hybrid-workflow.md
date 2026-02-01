# Hybrid Python-Julia Workflow

This guide explains how to combine BRIDGE.py and BRIDGE.jl for optimal performance by running GPU-accelerated stages in Python and CPU-bound stages in Julia.

## When to Use Hybrid Workflows

Use a hybrid approach when:

- You have GPU(s) available for Core 1 & 2
- You want faster Core 3 hierarchical linking
- You're processing large datasets where every stage matters
- You want maximum throughput from heterogeneous compute

## Prerequisites

### Python Environment

```bash
# RAPIDS for GPU acceleration
conda create -n bridge python=3.10
conda activate bridge
conda install -c rapidsai -c conda-forge -c nvidia \
    cupy cudf cugraph cuda-version=12.0

pip install bridge-tractography
```

### Julia Environment

```julia
using Pkg
Pkg.add([
    "JSON", "Dates", "Statistics", "LinearAlgebra",
    "Images", "ImageFiltering", "Graphs", "DataStructures",
    "SuiteSparseGraphBLAS"
])

# Add BRIDGE.jl (adjust path/URL as needed)
Pkg.develop(path="/path/to/BRIDGE.jl")
```

## Basic Hybrid Pipeline

### Step 1: Run GPU Stages in Python

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    "/data/input",
    "/data/project",
    sub_id="01",
    resolution=(1.0, 1.0, 1.0),
    
    # GPU stages
    run_core1=True,
    run_core2=True,
    
    # Skip CPU stage (will run in Julia)
    run_core3=False,
    
    # Core 1 params
    chunk_size=(512, 512, 512),
    sampling_density=0.0001,
    min_fa=0.3,
    
    # Core 2 params
    max_edge_distance=60,
    segment_radius=45,
)
```

### Step 2: Run CPU Stage in Julia

```julia
using BRIDGE

run_BRIDGE_setup_jl(
    "/data/input",
    "/data/project",
    sub_id = "01",
    
    # Skip GPU stages (already done)
    run_core1 = false,
    run_core2 = false,
    
    # Run Julia-accelerated Core 3
    run_core3 = true,
    overwrite = false,
)
```

### Step 3: Export Streamlines (Either Language)

**Julia (threaded, faster for large exports):**

```julia
using BRIDGE

BRIDGE_streamlines(
    "/data/project",
    sub_id = "01",
    min_fa = 0.25,
    min_length = 10.0,
    batch_size = 5000,
)
```

**Python (if you need nibabel/DIPY integration):**

```python
from bridge import BRIDGE_streamlines

BRIDGE_streamlines(
    "/data/project",
    sub_id="01",
    min_fa=0.25,
    min_length=10.0,
)
```

## Shell Script Example

A complete hybrid pipeline as a shell script:

```bash
#!/bin/bash
set -e

PROJECT="/data/project"
INPUT="/data/input"
SUB="01"

echo "=== BRIDGE Hybrid Pipeline ==="

# Step 1: GPU stages in Python
echo "[1/3] Running Core 1 & 2 (Python/GPU)..."
python << EOF
from bridge import BRIDGE_setup
BRIDGE_setup(
    "$INPUT", "$PROJECT",
    sub_id="$SUB",
    run_core1=True, run_core2=True, run_core3=False,
    chunk_size=(512, 512, 512),
    sampling_density=0.0001,
    min_fa=0.3,
    max_edge_distance=60,
    segment_radius=45,
)
EOF

# Step 2: CPU stage in Julia
echo "[2/3] Running Core 3 (Julia/CPU)..."
julia -t auto << EOF
using BRIDGE
run_BRIDGE_setup_jl(
    "$INPUT", "$PROJECT",
    sub_id="$SUB",
    run_core1=false, run_core2=false, run_core3=true,
)
EOF

# Step 3: Export streamlines
echo "[3/3] Exporting streamlines (Julia)..."
julia -t auto << EOF
using BRIDGE
BRIDGE_streamlines("$PROJECT", sub_id="$SUB", min_fa=0.25)
EOF

echo "=== Pipeline Complete ==="
```

## Shared Configuration File

Both implementations read the same JSON config:

```json
{
  "sub_id": "01",
  "desc": "brain",
  "resolution": [1.0, 1.0, 1.0],
  "overwrite": true,
  
  "core1": {
    "chunk_size": [512, 512, 512],
    "radius": 32,
    "sampling_density": 0.0001,
    "rho": 5,
    "sigma": 3,
    "min_fa": 0.3,
    "mask_type": "z"
  },
  
  "core2": {
    "max_edge_distance": 60,
    "edge_allignment_limit": 60,
    "segment_radius": 45,
    "segment_max_angle": 45,
    "min_component_size": 5
  },
  
  "run_core1": true,
  "run_core2": true,
  "run_core3": true
}
```

**Python:**
```python
BRIDGE_setup(input_dir, project_dir, config="config.json", run_core3=False)
```

**Julia:**
```julia
run_BRIDGE_setup_jl(input_dir, project_dir, config="config.json", run_core1=false, run_core2=false)
```

## Data Compatibility

Both implementations produce identical outputs:

| Data | Location | Format |
|------|----------|--------|
| Nodes | `network/nodes/*.bin` | Binary (37 bytes/node) |
| Layer 0 graphs | `network/hierarchical_graphs/lvl_0/` | Zarr-like JSON |
| Higher layers | `network/hierarchical_graphs/lvl_N/` | Zarr-like JSON |
| Streamlines | `outputs/tracts_*/` | TrackVis (.trk) |

You can freely mix:

- Python Core 1 → Julia Core 2 (when Julia Core 2 is complete)
- Python Core 1+2 → Julia Core 3
- Either implementation for streamlines

## Troubleshooting

### Julia Can't Find Python's Output

Ensure paths match exactly. Check the derivatives structure:

```bash
ls -la /data/project/derivatives/BRIDGE/sub-01/network/
# Should show: nodes/, hierarchical_graphs/
```

### Threading Issues

Set Julia threads explicitly:

```bash
export JULIA_NUM_THREADS=8
julia your_script.jl
```

### Memory Issues in Core 3

Julia's Core 3 uses less memory than Python. If you hit limits in Python Core 1/2, try reducing chunk size or processing fewer chunks in parallel.

## Performance Tips

1. **Match Julia threads to physical cores** — Don't exceed physical core count
2. **Use SSDs** — Component I/O is significant in Core 3
3. **Process sequentially by layer** — Julia does this automatically
4. **Batch streamline exports** — Use `batch_size` to control memory

## See Also

- [BRIDGE.jl Overview](../explanation/julia-implementation.md)
- [Julia API Reference](../reference/julia-api.md)
- [Configuration Options](../reference/config-options.md)
