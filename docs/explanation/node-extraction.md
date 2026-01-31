# Node Extraction

This document explains how BRIDGE extracts candidate fiber nodes from 3D image volumes.

## Overview

Node extraction identifies voxels likely to contain fiber bundles using local structural analysis. Two methods are supported:

1. **Structure tensor analysis** — For raw intensity images
2. **ODF peak detection** — For pre-computed orientation distribution functions

## Structure Tensor Method

### What is a Structure Tensor?

The structure tensor captures local orientation by analyzing intensity gradients:

```
S = Gρ * (∇I ⊗ ∇I)
```

Where:
- ∇I is the image gradient
- ⊗ is the outer product
- Gρ is Gaussian smoothing at scale ρ

The eigenvectors of S indicate local orientation. For fiber-like structures:
- λ₁ ≈ λ₂ >> λ₃ (two large, one small eigenvalue)
- The eigenvector for λ₃ points along the fiber

### Fractional Anisotropy

FA measures how directional the local structure is:

```
FA = √(3/2) × √(Σ(λᵢ - λ̄)²) / √(Σλᵢ²)
```

High FA (close to 1) indicates strong directionality — likely a fiber.
Low FA (close to 0) indicates isotropy — background or crossing region.

### Processing Steps

1. **Load chunk with padding** — Ensures boundary voxels have context
2. **Compute local z-score** — Normalizes for intensity variations
3. **Calculate structure tensor** — 3×3 tensor per voxel
4. **Eigendecomposition** — Extract eigenvalues and eigenvectors
5. **Compute FA** — Identify candidate fiber voxels
6. **Sample nodes** — Randomly subsample for efficiency

## ODF Peak Detection

For pre-computed ODFs (e.g., from CSD):

1. **Load spherical harmonic coefficients**
2. **Convert SH → ODF samples** on a sphere
3. **Compute GFA** (generalized FA)
4. **Find ODF peaks** — Local maxima on sphere
5. **Create nodes** — One per peak per sampled voxel

Multiple peaks per voxel handle fiber crossings.

## Sampling Strategy

### Why Sample?

Dense node extraction (every voxel) creates too many nodes for efficient graph construction. Sampling reduces this while preserving fiber topology.

### Sampling Parameters

- **sampling_density** — Fraction of valid voxels to sample
- **min_fa** — Threshold for valid voxels

Typical values:
- High-resolution (20µm): density=0.005, min_fa=0.3
- Standard dMRI (1mm): density=0.1, min_fa=0.2

## See Also

- [Core 1 API](../reference/api/core1.md)
- [Configuration Options](../reference/config-options.md)
