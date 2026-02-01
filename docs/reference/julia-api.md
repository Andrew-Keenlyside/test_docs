# Julia API Reference

This page documents the public API for BRIDGE.jl, the Julia implementation of the BRIDGE tractography pipeline.

## Pipeline Entry Points

### `run_BRIDGE_setup_jl`

Main pipeline orchestration function, equivalent to `BRIDGE_setup` in Python.

```julia
run_BRIDGE_setup_jl(
    input_dir::AbstractString,
    project_dir::AbstractString;
    config = nothing,
    sub_id = nothing,
    desc = nothing,
    resolution = nothing,
    rotation = nothing,
    overwrite = true,
    run_core1 = nothing,
    run_core2 = nothing,
    run_core3 = nothing,
    # Core 1 parameters
    chunk_size = nothing,
    radius = nothing,
    sampling_density = nothing,
    rho = nothing,
    sigma = nothing,
    min_fa = nothing,
    # Core 2 parameters
    max_edge_distance = nothing,
    edge_allignment_limit = nothing,
    segment_radius = nothing,
    # ...additional kwargs
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `input_dir` | `String` | required | Path to input data directory |
| `project_dir` | `String` | required | Path to BIDS project root |
| `config` | `String` or `Dict` | `nothing` | Path to JSON config or config Dict |
| `sub_id` | `String` | `"default"` | Subject identifier |
| `resolution` | `Vector{Float64}` | `[1.0, 1.0, 1.0]` | Voxel size in mm |
| `run_core1` | `Bool` | `true` | Execute node extraction |
| `run_core2` | `Bool` | `true` | Execute local graph construction |
| `run_core3` | `Bool` | `true` | Execute hierarchical linking |
| `overwrite` | `Bool` | `true` | Overwrite existing outputs |

**Example:**

```julia
using BRIDGE

run_BRIDGE_setup_jl(
    "/data/raw/sub-01",
    "/data/project",
    config = "config.json",
    sub_id = "01",
    run_core1 = false,  # Skip (use Python GPU version)
    run_core2 = false,  # Skip (use Python GPU version)  
    run_core3 = true    # Run Julia accelerated version
)
```

---

### `process_hierarchical_network`

Core 3 hierarchical linking, accelerated with SuiteSparseGraphBLAS.

```julia
process_hierarchical_network(
    directory::AbstractString;
    sub_id::AbstractString = "01",
    res::AbstractString = "1",
    start_layer::Union{Int,Nothing} = nothing,
    overwrite::Bool = false
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `directory` | `String` | required | Path to BRIDGE derivatives directory |
| `sub_id` | `String` | `"01"` | Subject identifier |
| `res` | `String` | `"1"` | Resolution label |
| `start_layer` | `Int` or `nothing` | `nothing` | Resume from specific layer |
| `overwrite` | `Bool` | `false` | Overwrite existing components |

**Example:**

```julia
using BRIDGE: process_hierarchical_network

# Process all layers
process_hierarchical_network(
    "/data/project/derivatives/BRIDGE/sub-01",
    sub_id = "01",
    res = "1"
)

# Resume from layer 2
process_hierarchical_network(
    "/data/project/derivatives/BRIDGE/sub-01",
    start_layer = 2,
    overwrite = true
)
```

---

### `extract_nodes`

Core 1 node extraction (CPU implementation).

```julia
extract_nodes(;
    bridge_path::AbstractString,
    volume::AbstractArray,
    chunk_size::NTuple{3,Int},
    radius::Int,
    sampling_density::Float64,
    rho::Real,
    sigma::Real,
    truncate::Real,
    mask_min::Real,
    mask_max::Real,
    min_fa::Real,
    sub_id::AbstractString,
    res::Real,
    mask_type::AbstractString = "z"
)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `bridge_path` | `String` | Output directory for BRIDGE derivatives |
| `volume` | `Array{T,3}` | 3D image volume |
| `chunk_size` | `NTuple{3,Int}` | Processing chunk dimensions |
| `radius` | `Int` | Local z-score kernel radius |
| `sampling_density` | `Float64` | Fraction of mask voxels to sample |
| `rho` | `Real` | Structure tensor integration scale |
| `sigma` | `Real` | Structure tensor differentiation scale |
| `min_fa` | `Real` | Minimum FA threshold |
| `sub_id` | `String` | Subject identifier |
| `res` | `Real` | Resolution in mm |

---

## Streamlines API

### `BRIDGE_streamlines`

Export network paths as streamlines.

```julia
BRIDGE_streamlines(
    directory::AbstractString;
    config_path = nothing,
    sub_id = nothing,
    bridge_label = "BRIDGE",
    origin_mask = nothing,
    terminus_mask = nothing,
    min_fa = nothing,
    max_fa = nothing,
    min_length = nothing,
    max_length = nothing,
    batch_size = nothing,
    per_region = false,
    freesurfer_path = nothing,
    # ...additional kwargs
)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `directory` | `String` | Project root directory |
| `sub_id` | `String` | Subject identifier |
| `bridge_label` | `String` | BRIDGE derivatives label |
| `origin_mask` | `Array` or `nothing` | Spatial mask for path origins |
| `terminus_mask` | `Array` or `nothing` | Spatial mask for path termini |
| `min_fa` | `Float64` | Minimum mean FA filter |
| `min_length` | `Float64` | Minimum path length filter |
| `batch_size` | `Int` | Paths per output file |
| `per_region` | `Bool` | Split output by region labels |

---

## Data Structures

### `Node`

Represents a single node in the tractography graph.

```julia
Base.@kwdef mutable struct Node
    id::Int32 = 0
    centre::NTuple{3,Int32} = (0, 0, 0)
    principal_eigenvector::NTuple{3,Float32} = (0.0, 0.0, 0.0)
    img::UInt8 = 0x00
    fa::Float32 = 0.0f0
    local_z::Float32 = 0.0f0
    is_endpoint::Bool = true
    searched::Bool = false
end
```

**Serialisation functions:**

```julia
# Binary format (interoperable with Python)
bytes = Node_to_binary(node)
node = Node_from_binary(bytes)

# Dict/JSON format
dict = Node_to_dict(node)
node = Node_from_dict(dict)
```

---

### `Graph`

Represents a graph component (chunk-level or merged).

```julia
Base.@kwdef mutable struct Graph
    id::String = ""
    nodes::Set{Any} = Set{Any}()
    paths::Dict{Any, Dict{String,Any}} = Dict()
    
    endpoints::Set{Any} = Set{Any}()
    start_paths::Dict{Any, Vector{Any}} = Dict()
    end_paths::Dict{Any, Vector{Any}} = Dict()
    path_endpoints::Dict{Any, Tuple{Any,Any}} = Dict()
    
    chunk_ids::Set{String} = Set{String}()
    centre_coords::Union{Vector{Float64},Nothing} = nothing
    
    layer::Int = 0
    lowest_layer::Bool = true
    subcomponents::Set{String} = Set{String}()
    linked_components::Dict{Any, Dict{Any, Vector{Any}}} = Dict()
end
```

**Key methods:**

```julia
# Serialisation
dict = Graph_to_dict(graph)
graph = Graph_from_dict(dict)

# Simplification (reduce path storage)
simplify!(graph, rebuild_indices=true)
```

The `simplify!` function reduces each path to `[start_id, path_id, end_id]` while preserving length and scores, then intersects nodes with endpoints.

---

## Utility Functions

### Component I/O

```julia
# Load all components grouped by layer
components = load_all_components_by_layer(graphs_dir)

# Load specific chunk components  
components = load_chunk_components(chunk_indices, layer_dir)

# Save component
save_component_zarr_like(component, output_dir)

# Find maximum hierarchy layer
max_layer = find_max_level(network_dir)
```

### Node I/O

```julia
# Get coordinates for multiple nodes
coords = find_node_coords(nodes_dir, node_ids)

# Get all node coordinates
all_coords = all_node_coords(nodes_dir)
```

### Path Utilities

```julia
# Extract full path from sparse representation
full_path = extract_subpath(path_id, component_dict)

# Get chunk ID from node ID
chunk_id = cid_from_node_id(node_id)

# Check if identifier is a node (vs path reference)
is_node(identifier)  # returns Bool
```

### Configuration

```julia
# Merge config with defaults
full_config = merge_with_defaults(defaults; config=user_config, overrides=cli_args)

# Parse BIDS-style filename
info = parse_bids_suffix("sub-01_res-1mm_cid-0-0-0_nodes.bin")
# Returns: Dict("sub"=>"01", "res"=>"1mm", "cid"=>"0-0-0")
```

---

## Threading Configuration

BRIDGE.jl uses Julia's native threading. Configure via:

```bash
# Set thread count
julia -t 8 script.jl

# Or via environment
export JULIA_NUM_THREADS=auto
```

BLAS threading is automatically restricted internally to prevent oversubscription:

```julia
ENV["OMP_NUM_THREADS"] = "1"
ENV["OPENBLAS_NUM_THREADS"] = "1"
ENV["MKL_NUM_THREADS"] = "1"
SuiteSparseGraphBLAS.set_nthreads(1)
```

---

## Dependencies

Required packages:

```julia
using Pkg
Pkg.add([
    "JSON", "Dates", "Statistics", "LinearAlgebra",
    "Images", "ImageFiltering", "Graphs", "DataStructures",
    "SuiteSparseGraphBLAS", "NVTX", "Serialization"
])

# Optional (visualisation)
Pkg.add(["GraphPlot", "Compose", "Colors", "Cairo", "Fontconfig"])
```

---

## See Also

- [Julia Implementation Overview](../explanation/julia-implementation.md) — Design rationale
- [Python API Reference](api-reference.md) — Equivalent Python functions
- [Configuration Options](config-options.md) — Shared configuration format
