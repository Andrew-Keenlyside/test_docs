# Pipeline Architecture

BRIDGE processes large 3D volumes through a three-stage pipeline that balances GPU memory constraints with the need for global connectivity analysis.

## Design Philosophy

### The Scale Problem

Connectomics images are large — a single mouse brain at 20µm resolution is approximately 500³ voxels (125 million). At higher resolutions used in electron microscopy, volumes can exceed terabytes. Processing such data requires:

1. **Chunked processing** — No single GPU can hold the entire volume
2. **Local-to-global reasoning** — Fibers span multiple chunks
3. **Efficient representation** — Can't store every voxel relationship

### BRIDGE's Solution

BRIDGE addresses this through hierarchical decomposition:

```
┌─────────────────────────────────────────┐
│           Global Network                │  Layer N
│         (unified graph)                 │
└─────────────────────────────────────────┘
                    ↑
          Hierarchical Merging (Core 3)
                    ↑
┌─────────┬─────────┬─────────┬─────────┐
│  Chunk  │  Chunk  │  Chunk  │  Chunk  │  Layer 1
│  Graph  │  Graph  │  Graph  │  Graph  │
└─────────┴─────────┴─────────┴─────────┘
                    ↑
          Local Graph Construction (Core 2)
                    ↑
┌─────────┬─────────┬─────────┬─────────┐
│  Nodes  │  Nodes  │  Nodes  │  Nodes  │  Layer 0
│ (chunk) │ (chunk) │ (chunk) │ (chunk) │
└─────────┴─────────┴─────────┴─────────┘
                    ↑
          Node Extraction (Core 1)
                    ↑
┌─────────────────────────────────────────┐
│              3D Volume                  │  Input
│           (NIfTI-Zarr)                  │
└─────────────────────────────────────────┘
```

## Stage Overview

### Core 1: Node Extraction (GPU)

**Input:** Chunked 3D image volume

**Process:**
1. Load chunk with boundary padding
2. Compute local z-score for intensity normalization
3. Calculate structure tensor for each voxel
4. Extract eigenvalues and eigenvectors
5. Compute fractional anisotropy (FA)
6. Sample candidate nodes from FA mask

**Output:** Binary node files with positions, orientations, and features

**Why GPU:** Structure tensor computation is embarrassingly parallel. Each voxel's tensor depends only on its local neighborhood.

### Core 2: Local Graph Construction (GPU)

**Input:** Node files from Core 1

**Process:**
1. Build spatial index (KD-tree) for neighbor lookup
2. Create initial edges based on proximity
3. Filter edges by geometric alignment
4. Remove triangle shortcuts
5. Define segments (node triplets)
6. Build segment graph
7. Apply minimum spanning forest
8. Extract paths via BFS

**Output:** Graph components with paths and quality scores

**Why GPU:** RAPIDS cuGraph provides GPU-accelerated graph algorithms. Edge computations parallelize well.

### Core 3: Hierarchical Merging (CPU)

**Input:** Chunk-level graph components

**Process:**
1. Identify boundary endpoints
2. Find cross-chunk endpoint matches
3. Link connected components
4. Merge and simplify paths
5. Repeat hierarchically until single network

**Output:** Unified brain-wide network

**Why CPU:** Hierarchical logic involves complex data structure manipulation. GraphBLAS provides efficient sparse operations on CPU.

## Data Flow

```
Image Chunks (NIfTI-Zarr)
      │
      ▼ [Per-chunk, GPU]
   ┌──────────────────┐
   │  Structure       │
   │  Tensor          │
   │  Analysis        │
   └──────────────────┘
      │
      ▼
   Node Files (.bin)
      │
      ▼ [Per-chunk with overlap, GPU]
   ┌──────────────────┐
   │  Edge Creation   │
   │  + Refinement    │
   │  + Segmentation  │
   │  + MSF Pruning   │
   │  + Path Search   │
   └──────────────────┘
      │
      ▼
   Graph Components (.zarr)
      │
      ▼ [Hierarchical, CPU]
   ┌──────────────────┐
   │  Boundary        │
   │  Linking         │
   │  + Merging       │
   │  + Simplification│
   └──────────────────┘
      │
      ▼
   Unified Network
      │
      ▼ [Export]
   Streamlines (.trk)
```

## Memory Management

### Chunking Strategy

Chunks are processed independently in Core 1 and Core 2:

- **Chunk size** controls GPU memory usage
- **Overlap** ensures boundary fibers aren't cut
- **Padding** provides context for structure tensor

Typical memory per chunk (256³ voxels):
- Input volume: ~67 MB (float32)
- With padding: ~130 MB
- Structure tensor intermediates: ~400 MB
- Total peak: ~600 MB per chunk

### Path Simplification

Higher hierarchy levels store simplified paths:

```python
# Layer 0: Full node sequence
["n1", "n2", "n3", "n4", "n5", "n6", "n7"]

# Layer 1: Endpoints + reference
["n1", "path_ref_L0_42", "n7"]

# Layer 2: Further simplified
["n1", "path_ref_L1_8", "n7"]
```

This keeps memory bounded regardless of path length.

## Parallelization

### Core 1 & 2: Dask + CUDA

- Dask distributes chunks across available GPUs
- Each worker processes independent chunks
- LocalCUDACluster manages GPU assignment

### Core 3: CPU Parallelism

- Layer-wise sequential processing (dependencies)
- Within-layer parallel chunk processing
- GraphBLAS for efficient sparse operations

## Design Decisions

### Why Not End-to-End Deep Learning?

1. **Interpretability** — Geometric constraints are explicit
2. **Generalization** — No training data required
3. **Scalability** — Classical algorithms scale predictably
4. **Debugging** — Each stage can be inspected

### Why Graphs Instead of Voxel Labels?

1. **Sparsity** — Only fiber voxels, not whole volume
2. **Topology** — Connections are first-class citizens
3. **Flexibility** — Easy to filter, query, export

### Why Minimum Spanning Forest?

1. **Parameter-free** — No thresholds to tune
2. **Preserves connectivity** — Guaranteed spanning
3. **Removes redundancy** — One path between points
4. **Efficient** — O(E log E) complexity

## See Also

- [Graph Construction](graph-construction.md) — Detailed algorithm explanation
- [Hierarchical Network](hierarchical-network.md) — Merging strategy details
- [Configuration Options](../reference/config-options.md) — Parameter tuning
