# Configure GPU Settings

This guide covers optimizing GPU configuration for BRIDGE.

## Check GPU Availability

```python
import cupy as cp

# List available GPUs
for i in range(cp.cuda.runtime.getDeviceCount()):
    props = cp.cuda.runtime.getDeviceProperties(i)
    print(f"GPU {i}: {props['name'].decode()}")
    print(f"  Memory: {props['totalGlobalMem'] / 1e9:.1f} GB")
    print(f"  Compute: {props['major']}.{props['minor']}")
```

## Memory Management

### Setting GPU Memory Limits

```python
from bridge import BRIDGE_setup

BRIDGE_setup(
    ...,
    GPU_limit=0.8  # Use 80% of GPU memory
)
```

### RMM Pool Allocation

<!-- Placeholder: Add RMM configuration when stabilized -->

For RAPIDS operations, configure the RMM memory pool:

```python
import rmm

# Pre-allocate 10GB pool
rmm.reinitialize(pool_allocator=True, initial_pool_size=10e9)
```

## Multi-GPU Setup

<!-- Placeholder: Add multi-GPU documentation -->

BRIDGE can utilize multiple GPUs via Dask-CUDA:

```python
from dask_cuda import LocalCUDACluster
from dask.distributed import Client

cluster = LocalCUDACluster()
client = Client(cluster)
```

## Troubleshooting

### CUDA Version Mismatch

Check that your CUDA toolkit matches RAPIDS requirements:

```bash
nvcc --version
nvidia-smi
```

### GPU Memory Fragmentation

If you encounter memory errors despite having free memory:

```python
import cupy as cp
cp.get_default_memory_pool().free_all_blocks()
```

## Next Steps

- [Run the Pipeline](run-pipeline.md)
- [Run on HPC Clusters](run-on-cluster.md)
