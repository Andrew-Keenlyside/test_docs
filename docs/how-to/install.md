# Install BRIDGE

This guide covers installing BRIDGE and all required dependencies.

## System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.10 | 3.11 |
| CUDA | 11.8 | 12.0+ |
| GPU | 8GB VRAM | 24GB+ VRAM |
| RAM | 32GB | 64GB+ |
| Storage | 100GB free | SSD recommended |

## Step 1: Install CUDA Toolkit

BRIDGE requires CUDA for GPU acceleration. Install the CUDA toolkit matching your GPU drivers:

```bash
# Check your current NVIDIA driver
nvidia-smi

# Install CUDA toolkit (Ubuntu example)
# Visit https://developer.nvidia.com/cuda-downloads for your OS
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get install cuda-toolkit-12-0
```

## Step 2: Create a Conda Environment

We recommend using conda to manage dependencies:

```bash
# Create environment with Python 3.11
conda create -n bridge python=3.11
conda activate bridge

# Install RAPIDS (includes cuDF, cuGraph, CuPy)
conda install -c rapidsai -c conda-forge -c nvidia \
    rapids=24.04 python=3.11 cuda-version=12.0
```

## Step 3: Install BRIDGE

### From PyPI (when available)

```bash
pip install bridge-tractography
```

### From Source

```bash
# Clone the repository
git clone https://github.com/Andrew-Keenlyside/BRIDGE.git
cd BRIDGE

# Install in development mode
pip install -e .
```

## Step 4: Install Additional Dependencies

```bash
# Core scientific packages
pip install numpy scipy nibabel networkx

# Dask for distributed computing
pip install dask distributed dask-cuda

# Structure tensor analysis
pip install structure-tensor

# Visualization (optional)
pip install matplotlib seaborn

# GraphBLAS for Core 3 (CPU stage)
pip install python-graphblas graphblas-algorithms
```

## Step 5: Verify Installation

```python
# Test imports
from bridge import BRIDGE_setup
from bridge.objs.Node import Node
from bridge.objs.Graph import Graph

# Test CUDA availability
import cupy as cp
print(f"CuPy version: {cp.__version__}")
print(f"CUDA available: {cp.cuda.is_available()}")

# Test RAPIDS
import cudf
import cugraph
print(f"cuDF version: {cudf.__version__}")
print(f"cuGraph version: {cugraph.__version__}")

print("Installation successful!")
```

## Troubleshooting

### "No module named 'cupy'"

RAPIDS installation failed. Try:
```bash
conda install -c rapidsai -c conda-forge cupy cuda-version=12.0
```

### "CUDA driver version is insufficient"

Your NVIDIA driver is too old. Update it:
```bash
sudo apt-get install nvidia-driver-535
sudo reboot
```

### Import errors with structure-tensor

Install the GPU version explicitly:
```bash
pip install structure-tensor[gpu]
```

### Memory errors during installation

Some RAPIDS packages are large. Use:
```bash
conda clean --all
conda install -c rapidsai rapids=24.04 --no-deps
```

## Platform-Specific Notes

### Windows

RAPIDS is not officially supported on Windows. Options:
1. Use WSL2 with Ubuntu
2. Use Docker with NVIDIA Container Toolkit
3. Run on a Linux HPC cluster

### macOS

CUDA is not available on macOS. BRIDGE requires a Linux system with NVIDIA GPUs.

### HPC Systems

See [Run on HPC Clusters](run-on-cluster.md) for module loading and environment setup on shared systems.

## Next Steps

- [Configure GPU Settings](configure-gpu.md)
- [Create a Configuration File](configure.md)
- [Run Your First Tractography](../tutorials/first-tractography.md)
