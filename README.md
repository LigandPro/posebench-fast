# posebench-fast

Fast and accurate docking evaluation metrics for molecular docking benchmarks.

[![PyPI version](https://badge.fury.io/py/posebench-fast.svg)](https://pypi.org/project/posebench-fast/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Why posebench-fast?

This is a **lightweight alternative** to the original [PoseBench](https://github.com/BioinfoMachineLearning/PoseBench) benchmark suite, focused on speed and practical metrics.

### posebench-fast vs PoseBench

| Feature | PoseBench (original) | posebench-fast |
|---------|---------------------|----------------|
| **Purpose** | Full benchmark suite with datasets | Evaluation metrics only |
| **Dependencies** | Heavy (DGL, ESM, etc.) | Minimal (rdkit, torch) |
| **Installation** | Complex setup | `pip install posebench-fast` |
| **Speed** | Full pipeline | 100x faster filters |
| **Use case** | Reproduce paper results | Integrate into your pipeline |

**When to use PoseBench:** Reproducing benchmark results, running full evaluation suite

**When to use posebench-fast:** Quick pose filtering, custom pipelines, production inference

---

Standard RMSD calculations can **severely penalize correct predictions** for symmetric molecules. Consider a benzene ring or a carboxylate group - rotating them by 180° gives chemically identical structures, but naive RMSD treats them as completely different poses.

### The Symmetry Problem

```
     O              O
     ‖              ‖
 R—C—O⁻   vs   ⁻O—C—R

 Same molecule, different atom ordering → High RMSD with naive calculation!
```

**posebench-fast** solves this by:
1. Computing all valid graph isomorphisms of the molecule
2. Finding the atom permutation that minimizes RMSD
3. Returning the **true** structural difference

### Fast vs Full PoseBusters

Full [PoseBusters](https://github.com/maabuu/posebusters) validation runs 27 tests including expensive energy calculations. For rapid screening of thousands of docking poses, this is too slow.

**posebench-fast** provides 4 key physics-based filters that catch most invalid poses in milliseconds:

| Filter | What it catches |
|--------|-----------------|
| `not_too_far_away` | Ligand flew away from binding site (distance > 5Å) |
| `no_clashes` | Atomic clashes with protein (overlapping VDW radii) |
| `no_volume_clash` | Volume overlap > 5% with protein |
| `no_internal_clash` | Invalid bond lengths/angles within ligand |

**Speed comparison:**
- Full PoseBusters: ~1-2 sec/pose
- posebench-fast: ~5-10 ms/pose (100x faster)

## Installation

```bash
pip install posebench-fast
```

Or with uv:
```bash
uv add posebench-fast
```

## Usage

### Symmetry-Corrected RMSD

```python
from posebench_fast import compute_all_isomorphisms, get_symmetry_rmsd_with_isomorphisms

# Compute isomorphisms once per molecule (cache this!)
isomorphisms = compute_all_isomorphisms(rdkit_mol)

# Compute symmetry-corrected RMSD for any number of predictions
rmsd = get_symmetry_rmsd_with_isomorphisms(true_coords, pred_coords, isomorphisms)

# Example: benzene with 12 equivalent atom orderings
# Naive RMSD: 2.4 Å (WRONG - penalizes valid rotation)
# SymmRMSD:   0.3 Å (CORRECT - finds best match)
```

### Fast PoseBusters Filters

```python
from posebench_fast import check_intermolecular_distance

results = check_intermolecular_distance(
    mol_orig=rdkit_mol,
    pos_pred=predicted_positions,    # (n_samples, n_atoms, 3)
    pos_cond=protein_positions,      # (n_protein_atoms, 3)
    atom_names_pred=ligand_atoms,    # ['C', 'N', 'O', ...]
    atom_names_cond=protein_atoms,   # ['C', 'CA', 'N', ...]
)

# Results dict contains:
# - not_too_far_away: [True, True, False, ...]  # ligand near protein?
# - no_clashes: [True, False, True, ...]        # no atomic overlaps?
# - no_volume_clash: [True, True, True, ...]    # no volume overlap?
# - no_internal_clash: [True, True, False, ...] # valid geometry?
# - is_buried_fraction: [0.8, 0.6, 0.2, ...]    # how buried is ligand?
```

### Batch Metrics with Filtering

```python
from posebench_fast import get_final_results_for_df, filter_results_by_fast

# Filter predictions to keep only physically valid ones
filtered = filter_results_by_fast(predictions)

# Compute benchmark metrics
metrics_df, scored = get_final_results_for_df(
    predictions,
    score_names=['error_estimate_0'],  # ranking score
    posebusters_filter=True,
    fast_filter=True
)

# metrics_df contains:
# | ranking          | RMSD<2Å | RMSD<5Å | SymRMSD<2Å | avg RMSD | ...
# |------------------|---------|---------|------------|----------|
# | error_estimate_0 | 0.45    | 0.78    | 0.52       | 3.2      |
# | ..._fast         | 0.48    | 0.82    | 0.58       | 2.9      |  <- after filtering
```

## API Reference

### RMSD Functions

| Function | Description |
|----------|-------------|
| `compute_all_isomorphisms(mol)` | Get all graph isomorphisms for symmetry correction |
| `get_symmetry_rmsd_with_isomorphisms(coords1, coords2, iso)` | Fast symmetry-corrected RMSD |
| `get_symmetry_rmsd(mol, coords1, coords2)` | Convenience wrapper (computes isomorphisms internally) |

### Filter Functions

| Function | Description |
|----------|-------------|
| `check_intermolecular_distance(...)` | All 4 fast filters + buried fraction |
| `check_volume_overlap(...)` | Volume overlap only |
| `check_geometry(...)` | Internal geometry validation only |

### Aggregation Functions

| Function | Description |
|----------|-------------|
| `get_final_results_for_df(...)` | Compute full metrics table |
| `filter_results_by_fast(...)` | Keep best poses by fast filter count |
| `filter_results_by_posebusters(...)` | Keep best poses by full PB count |
| `get_best_results_by_score(...)` | Select best pose per molecule by score |

## When to Use What

| Scenario | Recommendation |
|----------|----------------|
| Quick screening of 1000s of poses | `check_intermolecular_distance` |
| Final benchmark reporting | Full PoseBusters + `get_final_results_for_df` |
| Symmetric molecules (rings, COO⁻, etc.) | Always use `get_symmetry_rmsd_*` |
| Speed-critical pipelines | `filter_results_by_fast` |

## Related Projects

- **[PoseBench](https://github.com/BioinfoMachineLearning/PoseBench)** - Original comprehensive benchmark suite for molecular docking. Use if you need full datasets and reproducible benchmark runs.
- [spyrmsd](https://github.com/RMeli/spyrmsd) - Symmetry-corrected RMSD algorithms
- [PoseBusters](https://github.com/maabuu/posebusters) - Full physical validity validation (27 tests)

## License

MIT
