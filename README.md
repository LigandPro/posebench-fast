# posecheck-fast

[![PyPI version](https://badge.fury.io/py/posecheck-fast.svg)](https://pypi.org/project/posecheck-fast/)
[![CI](https://github.com/LigandPro/posecheck-fast/actions/workflows/ci.yml/badge.svg)](https://github.com/LigandPro/posecheck-fast/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)

Fast docking evaluation metrics: symmetry-corrected RMSD and lightweight PoseBusters filters.

## Installation

```bash
uv pip install posecheck-fast
```

## Features

- **Symmetry-corrected RMSD** — accounts for molecular symmetry (benzene, carboxylates, etc.)
- **Fast PoseBusters filters** — 4 key checks in ~10ms instead of full 27-test suite (~1-2s)

## Performance

**Up to ~4600x faster than full PoseBusters** on large pose batches, with optional GPU acceleration via PyTorch.

Benchmarked on **Kolmogorov** (AMD EPYC 9274F, 6x RTX 4090), **PDB 6LU7**, ligand from SMILES `CCNOC`.

| Poses (N) | PoseBusters (full) | posecheck-fast (CPU) | posecheck-fast (GPU) | Speedup vs PoseBusters |
|---:|---:|---:|---:|---:|
| 128 | 141.17 ms/pose (median) | 0.120 ms/pose (median) | 0.423 ms/pose (median) | 1175x (CPU), 334x (GPU) |
| 2048 | 157.58 ms/pose (1 run) | 0.0841 ms/pose (median) | 0.0342 ms/pose (median) | 1874x (CPU), 4608x (GPU) |

Notes:
- PoseBusters runs the full suite; posecheck-fast runs a small, fast subset of checks.
- GPU can be slower for small N due to overhead; it shines at larger batch sizes.

## Usage

```python
from posecheck_fast import compute_all_isomorphisms, get_symmetry_rmsd_with_isomorphisms

# Symmetry-corrected RMSD
isomorphisms = compute_all_isomorphisms(rdkit_mol)
rmsd = get_symmetry_rmsd_with_isomorphisms(true_coords, pred_coords, isomorphisms)
```

```python
from posecheck_fast import check_intermolecular_distance

# Fast filters: not_too_far_away, no_clashes, no_volume_clash, no_internal_clash
results = check_intermolecular_distance(
    mol_orig=rdkit_mol,
    pos_pred=pred_positions,      # (n_samples, n_atoms, 3)
    pos_cond=protein_positions,   # (n_protein_atoms, 3)
    atom_names_pred=lig_atoms,
    atom_names_cond=prot_atoms,
)
```

## Related

- [PoseBench](https://github.com/BioinfoMachineLearning/PoseBench) — full benchmark suite
- [PoseBusters](https://github.com/maabuu/posebusters) — full 27-test validation
- [spyrmsd](https://github.com/RMeli/spyrmsd) — symmetry RMSD algorithms

## License

MIT
