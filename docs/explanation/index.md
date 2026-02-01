# Explanation

This section provides in-depth understanding of BRIDGE's design, algorithms, and implementation decisions. These documents explain *why* things work the way they do.

## Core Concepts

| Document | Description |
|----------|-------------|
| [Pipeline Architecture](architecture.md) | Overall system design: chunking strategy, three-core pipeline, data flow |
| [Node Extraction](node-extraction.md) | Structure tensor theory, FA computation, stochastic sampling |
| [Graph Construction](graph-construction.md) | Edge creation, alignment filtering, segment definition, MSF pruning |
| [Hierarchical Network](hierarchical-network.md) | Octree structure, boundary linking, sparse path representation |
| [Pruning Strategies](pruning-strategies.md) | Parameter-free philosophy, MSF theory, state-aware extension |

## Implementations

BRIDGE is available in two language implementations that share identical algorithms and data formats:

| Document | Description |
|----------|-------------|
| [BRIDGE.jl (Julia)](julia-implementation.md) | Julia implementation for CPU acceleration of Core 3 and streamlines |

## Reading Guide

**New to BRIDGE?** Start with [Pipeline Architecture](architecture.md) for the big picture, then read the core concept documents in order.

**Choosing an implementation?** Read [BRIDGE.jl](julia-implementation.md) to understand when to use Python vs Julia.

**Tuning parameters?** The core concept documents explain what each parameter controls and why.

**Writing a paper?** These documents provide the algorithmic details and citations needed for methods sections.

## Implementation Summary

| Stage | BRIDGE.py | BRIDGE.jl |
|-------|-----------|-----------|
| Core 1 (Nodes) | GPU via CuPy | CPU via ImageFiltering.jl |
| Core 2 (Graphs) | GPU via cuGraph | In progress |
| Core 3 (Hierarchy) | CPU | CPU with GraphBLAS (faster) |
| Streamlines | Sequential | Multi-threaded (faster) |

Both implementations read and write identical file formats, enabling hybrid workflows where GPU stages run in Python and CPU stages run in Julia.
