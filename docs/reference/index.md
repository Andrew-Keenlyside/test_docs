# Reference

Technical specifications for the BRIDGE pipeline, including API documentation, configuration options, and file format specifications.

## API Reference

### Python (BRIDGE.py)

| Module | Description |
|--------|-------------|
| [Core 1: Node Extraction](api/core1.md) | `extract_nodes()` — GPU-accelerated structure tensor analysis |
| [Core 2: Graph Construction](api/core2.md) | `build_local_graphs()` — GPU-accelerated edge creation and MSF |
| [Core 3: Hierarchical Network](api/core3.md) | `process_hierarchical_network()` — CPU boundary linking |
| [Streamlines](api/streamlines.md) | `BRIDGE_streamlines()` — Path export to TrackVis format |
| [Network Queries](api/pull-network.md) | `BRIDGE_pull_network()` — Graph extraction and export |

### Julia (BRIDGE.jl)

| Module | Description |
|--------|-------------|
| [Julia API Reference](julia-api.md) | Complete Julia function reference |

## Configuration

| Document | Description |
|----------|-------------|
| [Configuration Options](config-options.md) | Complete parameter reference for `config.json` |
| [Default Values](defaults.md) | Default settings and recommended values |

## File Formats

| Format | Description |
|--------|-------------|
| [Node Binary Format](formats/nodes.md) | `.bin` files storing extracted nodes |
| [Component Format](formats/components.md) | Zarr-like JSON storage for graph components |
| [NIfTI-Zarr](formats/niftizarr.md) | Chunked image volume format |

## Quick Comparison: Python vs Julia

| Feature | Python | Julia |
|---------|--------|-------|
| **Import** | `from bridge import ...` | `using BRIDGE: ...` |
| **Main entry** | `BRIDGE_setup()` | `run_BRIDGE_setup_jl()` |
| **Core 3** | `process_hierarchical_network()` | `process_hierarchical_network()` |
| **Streamlines** | `BRIDGE_streamlines()` | `BRIDGE_streamlines()` |
| **Config format** | JSON | JSON (identical) |
| **Node format** | Binary (37 bytes) | Binary (37 bytes, identical) |

Both implementations are fully interoperable—outputs from one can be read by the other.

!!! tip "Choosing an Implementation"
    - **GPU available**: Use Python for Core 1 & 2 (RAPIDS acceleration)
    - **CPU-only or Core 3**: Julia offers ~3× speedup for hierarchical linking
    - **Hybrid**: Run Core 1-2 in Python, Core 3 in Julia for best performance
    
    See [Hybrid Workflow Guide](../how-to/hybrid-workflow.md) for details.

!!! note "API Stability"
    BRIDGE is under active development. APIs may change between versions. Pin your version in production environments.
