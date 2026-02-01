# BRIDGE.jl — Julia Implementation

BRIDGE is available in two language implementations: the primary **BRIDGE.py** (Python) and **BRIDGE.jl** (Julia). Both follow identical algorithmic principles, share data structures, and produce fully interoperable outputs—allowing hybrid workflows that leverage the strengths of each language.

## Why Two Implementations?

The BRIDGE pipeline has distinct computational profiles at each stage:

**Core 1 & 2** are GPU-bound. Structure tensor computation, edge filtering, and minimum spanning forest benefit enormously from GPU parallelism. Python's RAPIDS ecosystem (CuPy, cuGraph, cuDF) provides mature, well-optimised libraries for these operations.

**Core 3** is CPU-bound with complex control flow. Hierarchical linking involves graph traversal, dictionary manipulation, and sparse matrix operations that map poorly to GPU kernels. Julia's native performance, efficient threading model, and GraphBLAS bindings offer significant speedups here.

**Streamline export** is I/O and memory bound. Path expansion and coordinate lookup benefit from Julia's efficient memory handling and native threading.

The dual implementation lets you use the best tool for each stage while maintaining full data compatibility.

## Implementation Comparison

| Aspect | BRIDGE.py | BRIDGE.jl |
|--------|-----------|-----------|
| **Language** | Python 3.10+ | Julia 1.9+ |
| **GPU acceleration** | CuPy, cuGraph, RAPIDS | CUDA.jl (planned) |
| **CPU acceleration** | NumPy, NetworkX | SuiteSparseGraphBLAS, native threading |
| **Core 1 (nodes)** | ✅ GPU via CuPy | ✅ CPU via ImageFiltering.jl |
| **Core 2 (graphs)** | ✅ GPU via cuGraph | 🚧 In progress |
| **Core 3 (hierarchy)** | ✅ CPU | ✅ CPU with GraphBLAS |
| **Streamlines** | ✅ Sequential | ✅ Multi-threaded |

## Interoperability

### Shared Binary Format

Both implementations use identical binary formats. The Node structure is packed identically:

```
# 37 bytes per node, little-endian
struct Node:
    id:          Int32           # 4 bytes
    centre:      3 × Int32       # 12 bytes (z, y, x)
    eigenvector: 3 × Float32     # 12 bytes
    img:         UInt8           # 1 byte
    fa:          Float32         # 4 bytes
    local_z:     Float32         # 4 bytes
    is_endpoint: Bool            # 1 byte
    searched:    Bool            # 1 byte
```

This means node files written by Python can be read by Julia and vice versa.

### Shared Component Format

Graph components use identical Zarr-like JSON storage. Both implementations serialise and deserialise using the same schema:

```json
{
  "id": "sub-01_res-1mm_lvl-0_cid-0-0-0",
  "nodes": ["cid-0-0-0_idx-1", "cid-0-0-0_idx-2"],
  "paths": {
    "path_001": {
      "path": ["cid-0-0-0_idx-1", "path_001", "cid-0-0-0_idx-2"],
      "length": 45.2,
      "scores": {"FA": 0.42, "intensity": 128}
    }
  },
  "endpoints": ["cid-0-0-0_idx-1", "cid-0-0-0_idx-2"],
  "chunk_ids": ["0-0-0"],
  "layer": 0
}
```

### Shared Configuration

The same JSON configuration format works for both implementations:

```json
{
  "sub_id": "01",
  "resolution": [1.0, 1.0, 1.0],
  "core1": {
    "chunk_size": [512, 512, 512],
    "sampling_density": 0.0001,
    "min_fa": 0.3
  },
  "core2": {
    "max_edge_distance": 60,
    "segment_radius": 45
  },
  "run_core3": true
}
```

## Julia-Specific Optimisations

### SuiteSparseGraphBLAS for Core 3

The Julia implementation uses GraphBLAS for sparse matrix operations during path merging. This provides significant speedups for the adjacency matrix manipulations in hierarchical linking:

```julia
using SuiteSparseGraphBLAS
const GrB = SuiteSparseGraphBLAS

# Single-threaded BLAS to avoid oversubscription
ENV["OMP_NUM_THREADS"] = "1"
GrB.gbset(:nthreads, 1)
```

### Native Threading

Julia's threading model parallelises endpoint search and link verification:

```julia
using Base.Threads

@threads for i in eachindex(search_components)
    # Process component endpoints in parallel
    comp = search_components[i]
    # ... endpoint detection logic
end
```

For streamline export, path resolution uses spawn/fetch:

```julia
tasks = [@spawn resolve_single_path(rec, component_dict) for rec in records]
results = [fetch(t) for t in tasks]
```

### Type-Stable Data Structures

Julia's type system enables efficient, allocation-free inner loops:

```julia
const PathHit = NamedTuple{
    (:local_path_id, :node_path, :high_lvl_path_ids, :global_side),
    Tuple{String, Vector{String}, Vector{String}, Symbol},
}

const PathSideKey = Tuple{String, Int}
```

## Module Structure

```
BRIDGE.jl/
├── src/
│   ├── BRIDGE.jl                    # Main module
│   ├── core/
│   │   ├── core1_nodes.jl           # Node extraction (CPU)
│   │   ├── core2_local_graphs.jl    # Local graphs (WIP)
│   │   └── core3_hierarchical_network.jl  # Hierarchical linking
│   ├── objs/
│   │   ├── Node.jl                  # Node struct + serialisation
│   │   └── Graph.jl                 # Graph/Component struct
│   ├── utils/
│   │   ├── utils_general.jl         # BIDS parsing, config
│   │   ├── utils_nodes.jl           # Node I/O
│   │   ├── utils_components.jl      # Component I/O
│   │   └── utils_network.jl         # Path utilities
│   ├── api/
│   │   ├── streamlines.jl           # Streamline export
│   │   ├── pull_network.jl          # Network queries
│   │   └── image_chunks.jl          # Volume extraction
│   └── run.jl                       # Pipeline orchestration
```

## Current Status

| Module | Status | Notes |
|--------|--------|-------|
| `Node.jl` | ✅ Complete | Binary-compatible with Python |
| `Graph.jl` | ✅ Complete | Full serialisation support |
| `core1_nodes.jl` | ✅ Complete | CPU-only via ImageFiltering.jl |
| `core2_local_graphs.jl` | 🚧 In progress | Structure defined |
| `core3_hierarchical_network.jl` | ✅ Complete | GraphBLAS accelerated |
| `streamlines.jl` | ✅ Complete | Threaded export |
| `run.jl` | ✅ Complete | Pipeline orchestration |

## Performance

Benchmarks on typical datasets show Core 3 speedups of approximately 3× compared to the Python implementation, primarily due to GraphBLAS sparse operations and reduced memory allocation overhead. Streamline export sees similar improvements from threading.

GPU stages (Core 1 & 2) remain faster in Python due to the mature RAPIDS ecosystem—use the hybrid workflow for best overall performance.

## See Also

- [Pipeline Architecture](architecture.md) — Design shared by both implementations
- [Hybrid Workflow Guide](../how-to/hybrid-workflow.md) — Using Python and Julia together
- [Julia API Reference](../reference/julia-api.md) — Function documentation
