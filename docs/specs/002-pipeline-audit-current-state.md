# SPEC-002: Pipeline Deep Audit - Current State Documentation

**Status:** Complete (with validated corrections)
**Created:** 2024-12-01
**Author:** Ray (with Claude)
**Last Validated:** 2024-12-01

## Executive Summary

This document is a comprehensive audit of every file in `abmelt_infer_pipeline/`. It documents what exists, what each file does, how they connect, and **critical bugs discovered during validation**. This serves as the foundation for future refactoring and comparison with the reference implementation.

---

## Critical Bugs Identified

> **These bugs were validated from first principles by reading the actual source code.**

### 1. SASA Slicing Bug (res_sasa.py:47-48) - CRITICAL

**File:** `src/res_sasa.py`, function `get_core_surface()`

**Code:**
```python
ns = traj_length - start  # e.g., 100 - 20 = 80
core = np.mean(data[: -ns * 100, ...], axis=0)
```

**Problem:** The slice `[: -ns * 100]` selects the **first `start` ns (equilibration)** instead of the **last `ns` ns (production)**. With a 100ns trajectory and `start=20`:
- `ns = 80` (intended production phase)
- `[: -8000]` selects frames 0-2000 = first 20ns (equilibration phase)

**Impact:** All SASA-based features (`all-temp-sasa_core_mean_k=20_eq=20`, etc.) are computed from the equilibration phase, which should be discarded.

**Fix:** Change to `data[-ns * 100 :, ...]` or `data[start * 100 :, ...]`

---

### 2. Duplicate S² Computation Loop (order_param.py:261-310) - PERFORMANCE

**File:** `src/order_param.py`, function `order_s2()`

**Problem:** The same expensive S² computation loop appears twice:
- Lines 261-285: First computation
- Lines 287-310: Identical duplicate (after "saving order parameter values...")

**Impact:** Doubles the execution time of the most computationally intensive step.

**Fix:** Remove the duplicate loop (lines 287-310, keeping only the CSV save on line 311).

---

### 3. Broken Import Paths (Systemic) - RUNTIME FAILURE

**Files:** `src/structure_prep.py:14`, `src/md_simulation.py:15`

**Code:**
```python
sys.path.append(str(Path(__file__).parent.parent.parent / "AbMelt" / "src"))
```

**Problem:** This path resolves to `<repo_root>/AbMelt/src/`, but:
- The directory `AbMelt/` does not exist
- The reference code is at `_AbMelt_reference/`
- Required modules (`structure.immune_builder`, `preprocess`) cannot be imported

**Impact:** Pipeline fails immediately on import unless environment is manually patched.

**Fix:** Either:
1. Create symlink `AbMelt → _AbMelt_reference`
2. Refactor to proper Python package imports
3. Update path to `_AbMelt_reference`

---

### 4. Hardcoded Magic Numbers - FRAGILITY

| Location | Value | Purpose |
|----------|-------|---------|
| `order_param.py:142` | `0.89` | Lipari-Szabo scaling factor |
| `order_param.py:151` | `100` | Frames per ns assumption |
| `compute_descriptors.py:591` | `10` | ps per frame assumption |
| `compute_descriptors.py:664-671` | `2`, `5` | Potential radius indices |

**Impact:** If trajectory timestep differs from assumptions, features will be computed from wrong time ranges.

---

### 5. Double pdb2gmx Call (md_simulation.py:159-181) - MINOR

**Problem:** `pdb2gmx` called twice:
1. First: input PDB → `processed.pdb`
2. Second: `processed.pdb` → `processed.gro`

**Impact:** Minor inefficiency. Could output `.gro` directly or use `trjconv`.

---

## Directory Structure Overview

```
abmelt_infer_pipeline/
├── configs/                    # YAML configuration files
│   ├── paper_config.yaml       # Production config (100ns simulation)
│   └── testing_config.yaml     # Fast testing config (2ns simulation)
├── envs/                       # Environment setup (PROBLEMATIC - multiple pyproject.toml)
│   ├── abmelt_md_simulation_env/
│   │   ├── README.md
│   │   ├── linux_cuda_machine_setup.md
│   │   ├── linux_cuda_machine_setup_1.sh
│   │   ├── linux_cuda_machine_setup_2.sh
│   │   ├── mac_env_setup.md
│   │   ├── poetry.lock
│   │   └── pyproject.toml      # ← SECOND pyproject.toml!
│   └── abmelt_sklearn_model_infer_env/
│       ├── mac_env_setup.md
│       ├── poetry.lock
│       └── pyproject.toml      # ← THIRD pyproject.toml!
├── mdp/                        # GROMACS MDP parameter files
│   ├── arr.sh                  # HPC array job script
│   ├── em.mdp                  # Energy minimization
│   ├── ions.mdp                # Ion addition
│   ├── md.mdp                  # Production MD
│   ├── npt.mdp                 # NPT equilibration
│   └── nvt.mdp                 # NVT equilibration
├── models/                     # Trained ML models
│   ├── README.md
│   ├── tagg/                   # Aggregation temperature
│   │   ├── efs_best_knn.pkl
│   │   └── rf_efs.csv
│   ├── tm/                     # Melting temperature
│   │   ├── efs_best_randomforest.pkl
│   │   └── rf_efs.csv
│   └── tmon/                   # Onset melting temperature
│       ├── efs_best_elasticnet.pkl
│       └── rf_efs.csv
├── run_data/                   # Runtime output (should be .gitignored)
│   ├── results/
│   │   └── predictions_summary.csv
│   └── run_2/temp/
├── src/                        # Core Python modules
│   ├── cleanup_temp_files.py
│   ├── compute_descriptors.py
│   ├── md_simulation.py
│   ├── model_inference.py
│   ├── order_param.py          # ⚠️ BUG: duplicate loop
│   ├── preprocess.py
│   ├── res_sasa.py             # ⚠️ BUG: inverted slice
│   ├── structure_prep.py       # ⚠️ BUG: broken import path
│   └── REQUIRED_FILES.md
├── tests/                      # Test scripts (NOT pytest-compatible)
│   ├── quick_test.py
│   ├── run_tests.py
│   ├── simple_structure_test.py
│   └── test_structure_generation.py
├── .gitignore
├── CLEANUP_GUIDE.md
├── demo_structure_generation.py
├── Dockerfile
├── feature_names.txt
├── infer.py                    # Main CLI entrypoint
├── infer_using_descriptors.py  # Alternative inference (standalone)
├── my_antibody.pdb             # Example input
└── README.md
```

---

## File-by-File Deep Audit

### Root Level Files

#### `infer.py` (342 lines) - Main CLI Entrypoint

**Purpose:** Main inference pipeline orchestrator. Takes antibody input (sequences or PDB) and runs the complete 4-step pipeline.

**Key Functions:**
- `main()` - CLI argument parsing and orchestration
- `load_config(path)` - YAML config loading
- `setup_logging(config)` - Logging configuration
- `create_directories(config)` - Output directory setup
- `run_inference_pipeline(antibody, config, skip_*)` - Pipeline execution

**Pipeline Steps:**
1. Structure preparation (ImmuneBuilder or PDB processing)
2. MD simulation (GROMACS)
3. Descriptor computation
4. Model inference (scikit-learn)

**CLI Interface:**
```bash
# From sequences
python infer.py --h "HEAVY_SEQ" --l "LIGHT_SEQ" --name "antibody" --config configs/paper_config.yaml

# From PDB
python infer.py --pdb "file.pdb" --name "antibody" --config configs/paper_config.yaml

# Skip flags available: --skip-structure, --skip-md, --skip-descriptors, --skip-inference
```

---

#### `infer_using_descriptors.py` (80 lines) - Standalone Descriptor Inference

**Purpose:** Alternative inference path that takes pre-computed descriptors and runs model prediction only.

**Key Functions:**
- `build_model_feature_col_map()` - Reads feature columns from CSV files
- `infer_using_descriptors(model_name, descriptor_file, feature_names)` - Model prediction

**⚠️ Issue:** Hardcoded reference to `AbMelt/data/` directory (line 44) - stale path.

---

#### `demo_structure_generation.py` (272 lines) - Demo Script

**Purpose:** Demonstration script showing how to use structure generation functionality.

**Key Functions:**
- `demo_sequence_based_generation()` - ImmuneBuilder demo
- `demo_pdb_processing()` - PDB file processing demo
- `demo_structure_analysis()` - BioPython structure analysis
- `cleanup_demo_files()` - Cleanup

**Note:** Should be moved to `examples/` directory.

---

#### `feature_names.txt` (22 lines) - Feature Documentation

**Content Summary:**
- **Tagg (3 features):** `rmsf_cdrs_mu_400`, `gyr_cdrs_Rg_std_400`, `all-temp_lamda_b=25_eq=20`
- **Tm (3 features):** `gyr_cdrs_Rg_std_350`, `bonds_contacts_std_350`, `rmsf_cdrl1_std_350`
- **Tmon (4 features):** `bonds_contacts_std_350`, `all-temp-sasa_core_mean_k=20_eq=20`, `all-temp-sasa_core_std_k=20_eq=20`, `r-lamda_b=2.5_eq=20`

---

#### `Dockerfile` (152 lines) - Container Build

**Key Components:**
1. Base image: `runpod/pytorch:1.0.1-cu1281-torch280-ubuntu2404`
2. Installs: Miniconda, Poetry, GROMACS 2024 (with CUDA)
3. Creates conda environment with: OpenMM, pdbfixer, anarci

**Build Args:**
- `CUDA_TARGET_COMPUTE=89` (default for RTX 4090)
- `REPO_URL` - GitHub repo to clone

---

### `src/` Directory - Core Modules

#### `src/structure_prep.py` (310 lines) - Structure Preparation

**Purpose:** Handles antibody structure generation and preprocessing.

**Key Functions:**
- `prepare_structure(antibody, config)` - Main entrypoint
- `_prepare_from_pdb(antibody, config)` - Process existing PDB
- `_prepare_from_sequences(antibody, config)` - Generate from sequences
- `validate_structure(pdb_file)` - Checks for 2+ chains
- `rename_chains_to_ab(pdb_file)` - Renames H/L to A/B using ANARCI
- `get_chain_sequences(pdb_file)` - Extracts sequences from PDB

**Dependencies:**
- BioPython (PDBParser, seq1)
- ANARCI (chain type identification)
- ImmuneBuilder (via `structure.immune_builder`)

**⚠️ Bug:** Line 14 uses `sys.path.append()` to import from `AbMelt/src/` which doesn't exist.

---

#### `src/preprocess.py` (524 lines) - GROMACS Preprocessing

**License:** MIT - Copyright Merck Sharp & Dohme Corp.

**Key Functions:**
- `parse_propka(pka)` - Parse PropKa output file
- `identify_chain_types(pdb_file)` - Use ANARCI to identify H/L chains
- `convert_pkas(pkas, pH, light_chain_id, heavy_chain_id)` - Extract histidine protonation states
- `protonation_state(pdb_filename, pdb_path, pH)` - Run PropKa and get GROMACS input
- `canonical_index(pdb)` - Create GROMACS index groups for CDRs using IMGT numbering
- `edit_mdp(mdp, new_mdp, extend_parameters, **substitutions)` - Modify MDP files

**IMGT CDR Definitions:**
| Region | Residues |
|--------|----------|
| CDR-L1 | 27-38 |
| CDR-L2 | 56-65 |
| CDR-L3 | 105-117 |
| CDR-H1 | 27-38 |
| CDR-H2 | 56-65 |
| CDR-H3 | 105-117 |

**Index Groups Created:**
| Group | Name |
|-------|------|
| 10 | light_chain |
| 11 | heavy_chain |
| 12-14 | cdrl1, cdrl2, cdrl3 |
| 15-17 | cdrh1, cdrh2, cdrh3 |
| 18 | cdrs (combined) |

---

#### `src/md_simulation.py` (930 lines) - MD Simulation

**Purpose:** Complete GROMACS MD simulation workflow.

**Key Functions:**
- `run_md_simulation(structure_files, config)` - Main entrypoint
- `_preprocess_for_gromacs(pdb_filename, pdb_filepath, config)` - pdb2gmx, indexing
- `_setup_simulation_system(gromacs_files, config)` - Box, solvation, ions, energy minimization
- `_run_multi_temp_simulations(system_files, config)` - Run at multiple temperatures
- `_process_trajectories(trajectory_files, config)` - PBC removal, final trajectory
- `setup_gromacs_environment(gromacs_path, mdp_dir)` - Configure GROMACS

**Simulation Workflow:**
1. **Preprocess:** pdb2gmx (topology, protonation), make_ndx (CDR groups)
2. **Setup:** editconf (box), solvate, genion (ions), energy minimization
3. **Equilibration:** NVT (temperature coupling), NPT (pressure coupling)
4. **Production:** MD at each temperature (300K, 350K, 400K)
5. **Post-process:** trjconv (remove PBC, center protein)

**⚠️ Bug:** Line 15 uses broken `sys.path.append()` for imports.
**⚠️ Minor:** Lines 159-181 call `pdb2gmx` twice.

---

#### `src/compute_descriptors.py` (985 lines) - Descriptor Computation

**Purpose:** Extract MD descriptors from trajectories and aggregate into ML-ready DataFrame.

**Key Functions:**
- `compute_descriptors(simulation_result, config)` - Main entrypoint
- `_compute_gromacs_descriptors(work_dir, temps, eq_time)` - Generate XVG files
- `_compute_order_parameters(...)` - S² calculation via `order_param.py`
- `_compute_core_surface_sasa(...)` - Core/surface SASA via `res_sasa.py`
- `_compute_lambda_features(...)` - Multi-temp lambda slope
- `_aggregate_descriptors_to_dataframe(...)` - Parse XVG files into DataFrame

**GROMACS Descriptors Computed:**
- **Global:** SASA, H-bonds, contacts, RMSD, gyration radius, dipole moment
- **Per-CDR:** SASA, RMSF, gyration radius, electrostatic potential
- **Inter-chain:** Light-heavy H-bonds
- **Entropy:** Conformational entropy via covariance analysis

**Feature Naming Convention:**
```
{metric}_{region}_{stat}_{temperature}
Examples:
  rmsf_cdrs_mu_400
  gyr_cdrs_Rg_std_350
  bonds_contacts_std_350
  all-temp_lamda_b=25_eq=20
```

**⚠️ Issue:** Line 591 assumes 10ps/frame without verification.

---

#### `src/model_inference.py` (372 lines) - ML Model Inference

**Purpose:** Load trained scikit-learn models and make predictions.

**Class:** `AbMeltPredictor`

**Model Definitions (Hardcoded):**
```python
model_paths = {
    "tagg": models_dir / "tagg" / "efs_best_knn.pkl",
    "tm": models_dir / "tm" / "efs_best_randomforest.pkl",
    "tmon": models_dir / "tmon" / "efs_best_elasticnet.pkl",
}

model_features = {
    "tagg": ["rmsf_cdrs_mu_400", "gyr_cdrs_Rg_std_400", "all-temp_lamda_b=25_eq=20"],
    "tm": ["gyr_cdrs_Rg_std_350", "bonds_contacts_std_350", "rmsf_cdrl1_std_350"],
    "tmon": ["bonds_contacts_std_350", "all-temp-sasa_core_mean_k=20_eq=20",
             "all-temp-sasa_core_std_k=20_eq=20", "r-lamda_b=2.5_eq=20"],
}
```

---

#### `src/order_param.py` (334 lines) - Order Parameter Calculation ⚠️ BUG

**License:** MIT - Copyright Merck Sharp & Dohme Corp.

**Purpose:** Calculate N-H bond order parameters (S²) and temperature-dependent lambda values.

**Key Functions:**
- `order_s2(mab, temp, block_length, start, use_dummy)` - Main S² calculation
- `get_h_n_coords(u)` - Extract H-N bond coordinates
- `get_vectors(coords_dict)` - Compute normalized bond vectors
- `get_s2(product_dict)` - Calculate S²: `S² = 0.89 * (1.5 * Σ(⟨uᵢuⱼ⟩²) - 0.5)`
- `get_lambda(master_dict, temps)` - Linear regression of log(1-√S²) vs log(T)

**⚠️ BUG:** Lines 261-310 contain a duplicate loop that recomputes S² unnecessarily, doubling execution time.

**⚠️ Magic Numbers:**
- Line 142: `0.89` (Lipari-Szabo scaling factor)
- Line 151: `block_length *= 100` (assumes 100 frames/ns)

---

#### `src/res_sasa.py` (68 lines) - Residue SASA Calculation ⚠️ CRITICAL BUG

**License:** MIT - Copyright Merck Sharp & Dohme Corp.

**Purpose:** Calculate per-residue SASA and identify core/surface residues.

**Key Functions:**
- `core_surface(temp)` - Compute residue SASA using mdtraj shrake_rupley
- `get_core_surface(sasa_dict, temp, k, start)` - Aggregate core/surface statistics
- `get_slope(data)` - Linear regression slope

**⚠️ CRITICAL BUG:** Lines 47-50 have inverted slice logic:
```python
ns = traj_length - start
core = np.mean(data[: -ns * 100, ...], axis=0)  # WRONG: selects equilibration
# Should be: data[-ns * 100 :, ...] or data[start * 100 :, ...]
```

---

#### `src/cleanup_temp_files.py` (382 lines) - Cleanup Utility

**Purpose:** Remove intermediate GROMACS files while preserving required files.

**CLI:**
```bash
python src/cleanup_temp_files.py run_data/run_2/temp \
    --antibody-name "my_antibody" \
    --temperatures 300 350 400 \
    --dry-run
```

---

### `models/` Directory - Trained Models

| Model | Algorithm | Features |
|-------|-----------|----------|
| tagg/efs_best_knn.pkl | KNN | rmsf_cdrs_mu_400, gyr_cdrs_Rg_std_400, all-temp_lamda_b=25_eq=20 |
| tm/efs_best_randomforest.pkl | Random Forest | gyr_cdrs_Rg_std_350, bonds_contacts_std_350, rmsf_cdrl1_std_350 |
| tmon/efs_best_elasticnet.pkl | ElasticNet | bonds_contacts_std_350, all-temp-sasa_core_mean_k=20_eq=20, all-temp-sasa_core_std_k=20_eq=20, r-lamda_b=2.5_eq=20 |

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ABMELT INFERENCE PIPELINE                        │
└─────────────────────────────────────────────────────────────────────────────┘

INPUT:
  ├── Sequences (--h/--l)  ─┐
  │                         ├──► structure_prep.py ──► {name}.pdb
  └── PDB file (--pdb)     ─┘

STEP 1: Structure Preparation (structure_prep.py)
  ├── ImmuneBuilder (if sequences)
  ├── Chain renaming (H/L → A/B)
  └── Validation

  OUTPUT: {name}.pdb, chains dict

STEP 2: MD Simulation (md_simulation.py)
  ├── preprocess.py:
  │   ├── PropKa (protonation states)
  │   ├── pdb2gmx (topology)
  │   └── make_ndx (CDR index groups)
  ├── System setup:
  │   ├── editconf (box)
  │   ├── solvate (water)
  │   ├── genion (ions)
  │   └── energy minimization
  └── MD runs:
      ├── NVT equilibration (per temp)
      ├── NPT equilibration (per temp)
      ├── Production MD (per temp)
      └── Trajectory processing (PBC removal)

  OUTPUT: md_final_{temp}.xtc, md_final_{temp}.gro, md_{temp}.tpr

STEP 3: Descriptor Computation (compute_descriptors.py)
  ├── GROMACS analysis:
  │   ├── sasa, rms, gyrate, hbond, rmsf
  │   ├── potential, dipoles, covar/anaeig
  │   └── *.xvg files
  ├── order_param.py:
  │   ├── S² per residue per temperature ⚠️ DUPLICATE LOOP
  │   └── Lambda (temperature slope)
  └── res_sasa.py:
      └── Core/surface SASA ⚠️ INVERTED SLICE

  OUTPUT: descriptors.csv, *.xvg files

STEP 4: Model Inference (model_inference.py)
  ├── Load models (tagg, tm, tmon)
  ├── Extract features
  └── Predict

  OUTPUT: {name}_predictions.csv
      ├── T_agg (°C)
      ├── T_m (°C)
      └── T_m_onset (°C)
```

---

## Feature Requirements Matrix

| Feature Name | Model(s) | Temperature | Source Module | Computation |
|--------------|----------|-------------|---------------|-------------|
| rmsf_cdrs_mu_400 | tagg | 400K | compute_descriptors | GROMACS rmsf |
| rmsf_cdrl1_std_350 | tm | 350K | compute_descriptors | GROMACS rmsf |
| gyr_cdrs_Rg_std_350 | tm | 350K | compute_descriptors | GROMACS gyrate |
| gyr_cdrs_Rg_std_400 | tagg | 400K | compute_descriptors | GROMACS gyrate |
| bonds_contacts_std_350 | tm, tmon | 350K | compute_descriptors | GROMACS hbond |
| all-temp_lamda_b=25_eq=20 | tagg | All | order_param | S² regression |
| r-lamda_b=2.5_eq=20 | tmon | All | order_param | S² regression R² |
| all-temp-sasa_core_mean_k=20_eq=20 | tmon | All | res_sasa ⚠️ | mdtraj SASA |
| all-temp-sasa_core_std_k=20_eq=20 | tmon | All | res_sasa ⚠️ | mdtraj SASA |

---

## Configuration Reference

```yaml
simulation:
  temperatures: [300, 350, 400]  # Required for multi-temp features
  simulation_time: 100           # ns, minimum ~30ns for lambda features
  force_field: "charmm27"
  water_model: "tip3p"
  salt_concentration: 150        # mM
  pH: 7.4
  p_salt: "NA"
  n_salt: "CL"
  gpu_enabled: true

paths:
  run_dir: "run_data/run_1/"
  output_dir: "results"
  temp_dir: "temp"
  log_dir: "logs"

gromacs:
  mdp_dir: "mdp"
  n_threads: 8
  gpu_id: 0

descriptors:
  equilibration_time: 20         # ns to skip at start
  block_length: [2.5, 25]        # ns for S² blocks
  core_surface_k: 20             # residues for core/surface
  compute_lambda: true
  use_dummy_s2: false            # true for testing only

performance:
  cleanup_temp: true
  cleanup_after: "inference"
  delete_order_params: false

logging:
  level: "INFO"
  file: "logs/inference.log"
```

---

## Dependencies

### Python Packages (from root pyproject.toml)

```
torch>=2.0.0
immunebuilder>=1.2
pyyaml>=6.0
gromacswrapper>=0.9.2
propka>=3.5.0
pandas>=2.0.0
joblib>=1.3.0
numpy>=1.23.0
mdanalysis>=2.6.0
scikit-learn==1.2.0    # Pinned version!
freesasa>=2.2.0
mdtraj>=1.9.9
biopython>=1.79
```

### Conda Packages (from Dockerfile)

```
openmm
pdbfixer
anarci
```

### External Software

- GROMACS 2024 (with CUDA)
- CHARMM force field files

---

## Known Issues Summary

### Critical (Affects Results)

1. **SASA slicing bug** (`res_sasa.py:47-48`) - Analyzes equilibration instead of production
2. **Duplicate S² loop** (`order_param.py:261-310`) - Doubles computation time

### Blocking (Runtime Failure)

3. **Broken import paths** (`structure_prep.py:14`, `md_simulation.py:15`) - Import fails

### Structural

4. **Multiple pyproject.toml files** in `envs/` - violates single source of truth
5. **No `__init__.py`** in `src/` - not a proper Python package
6. **`sys.path.append()` hacks** for imports
7. **Tests not pytest-compatible** - custom test framework
8. **Files in wrong locations:**
   - `my_antibody.pdb` → tests/fixtures/
   - `demo_structure_generation.py` → examples/
   - `CLEANUP_GUIDE.md` → docs/how-to/
   - `src/REQUIRED_FILES.md` → docs/

### Minor

9. **Hardcoded magic numbers** (frame rates, scaling factors)
10. **Double pdb2gmx call** in `md_simulation.py`
11. **Minimal .gitignore** - should include run_data/, *.pdb outputs
12. **scikit-learn pinned to 1.2.0** - may cause compatibility issues

---

## Summary Statistics

| Metric | Count |
|--------|-------|
| Total Python files | 13 |
| Total lines of Python | ~4,700 |
| Critical bugs | 2 |
| Blocking issues | 1 |
| Config files | 2 |
| MDP files | 5 (+1 shell script) |
| Model files | 6 (3 .pkl + 3 .csv) |

---

## Next Steps

This audit provides the foundation for:
1. **Immediate:** Fix critical bugs (SASA slice, S² duplicate)
2. **Phase 2 (Cleanup):** Fix imports, delete pycache, nested configs
3. **Phase 3 (Restructure):** Move files to proper locations, add __init__.py
4. **Comparison:** Compare with `_AbMelt_reference/` for additional discrepancies
5. **Validation:** Compare with paper methodology in `reference_literature/markdown/abmelt.md`
