# SPEC-001: Pipeline Refactoring Plan

**Status:** Draft
**Created:** 2024-12-01
**Author:** Ray

## Executive Summary

Refactor `abmelt_infer_pipeline/` from Praful's research-quality code into a canonical, production-ready ML inference package. This spec inventories every file, identifies issues, and proposes a clean structure.

---

## Current State Inventory

### Root Level: `abmelt_infer_pipeline/`

| File | Purpose | Issues | Action |
|------|---------|--------|--------|
| `infer.py` | Main inference entrypoint | Needs review | Keep, refactor |
| `infer_using_descriptors.py` | Alt inference path | Redundant? | Review/merge |
| `demo_structure_generation.py` | Demo script | Should be in examples/ | Move |
| `feature_names.txt` | Feature list for models | Should be in models/ or configs/ | Move |
| `my_antibody.pdb` | Example input | Should be in examples/ or tests/fixtures | Move |
| `Dockerfile` | Container build | Review for correctness | Keep |
| `README.md` | Package docs | Needs update after refactor | Update |
| `CLEANUP_GUIDE.md` | Temp file cleanup guide | Merge into README or how-to | Consolidate |
| `.gitignore` | Git ignores | Redundant with root .gitignore? | Review |

### `configs/`

| File | Purpose | Issues | Action |
|------|---------|--------|--------|
| `paper_config.yaml` | Config matching paper params | Good | Keep |
| `testing_config.yaml` | Fast testing config | Good | Keep |

**Assessment:** Structure OK. May need schema validation.

### `envs/` (PROBLEMATIC)

```
envs/
├── abmelt_md_simulation_env/
│   ├── README.md
│   ├── linux_cuda_machine_setup.md
│   ├── linux_cuda_machine_setup_1.sh
│   ├── linux_cuda_machine_setup_2.sh
│   ├── mac_env_setup.md
│   ├── poetry.lock
│   └── pyproject.toml          # ← SECOND pyproject.toml!
└── abmelt_sklearn_model_infer_env/
    ├── mac_env_setup.md
    ├── poetry.lock
    └── pyproject.toml          # ← THIRD pyproject.toml!
```

**Issues:**
1. Multiple `pyproject.toml` files - violates single-source-of-truth
2. Poetry lock files when we use uv at root
3. Embedded environment configs - should be in docs/how-to/
4. Platform-specific shell scripts - move to scripts/ or docs/

**Action:**
- Delete nested pyproject.toml/poetry.lock files
- Move setup guides to `docs/how-to/`
- Move shell scripts to `scripts/` if still needed
- Delete `envs/` directory entirely

### `mdp/` (GROMACS config files)

| File | Purpose | Issues | Action |
|------|---------|--------|--------|
| `em.mdp` | Energy minimization params | Good | Keep |
| `ions.mdp` | Ion addition params | Good | Keep |
| `nvt.mdp` | NVT equilibration | Good | Keep |
| `npt.mdp` | NPT equilibration | Good | Keep |
| `md.mdp` | Production MD params | Good | Keep |
| `arr.sh` | Array job script? | Review - may be HPC specific | Review |

**Assessment:** Core MD parameters - keep but verify against paper.

### `models/`

```
models/
├── README.md
├── tagg/
│   ├── efs_best_knn.pkl
│   └── rf_efs.csv
├── tm/
│   ├── efs_best_randomforest.pkl
│   └── rf_efs.csv
└── tmon/
    ├── efs_best_elasticnet.pkl
    └── rf_efs.csv
```

**Assessment:** Clean structure. These are the trained model weights from the paper.
- `.pkl` files: scikit-learn model weights
- `.csv` files: feature importance/selection data

**Action:** Keep as-is. Verify pkl files match `_AbMelt_reference/models/`.

### `run_data/` (RUNTIME OUTPUT)

```
run_data/
├── results/
│   └── predictions_summary.csv
└── run_2/
    └── temp/
```

**Issues:**
1. Contains runtime output - should be in .gitignore
2. `run_2/` suggests manual run naming - should be automated

**Action:**
- Add to .gitignore (output dir, not committed)
- Document expected output structure

### `src/` (CORE CODE)

| File | Purpose | Issues | Action |
|------|---------|--------|--------|
| `structure_prep.py` | PDB preparation, ImmuneBuilder | Core - review | Audit |
| `preprocess.py` | GROMACS preprocessing | Core - review | Audit |
| `md_simulation.py` | MD simulation runner | Core - review | Audit |
| `compute_descriptors.py` | Descriptor calculation | Core - review | Audit |
| `model_inference.py` | ML model inference | Core - review | Audit |
| `order_param.py` | S² order parameter calc | From AbMelt - review | Audit |
| `res_sasa.py` | Residue SASA calculation | From AbMelt - review | Audit |
| `cleanup_temp_files.py` | Temp file cleanup | Utility | Keep |
| `REQUIRED_FILES.md` | Doc in src/ | Move to docs/ | Move |
| `__pycache__/` | Python cache | Should be gitignored | Delete |

**Issues:**
1. No `__init__.py` - not a proper Python package
2. `__pycache__` committed - should be ignored
3. Documentation mixed with code

**Action:**
- Add `__init__.py` with proper exports
- Delete `__pycache__/`
- Move docs to `docs/`

### `tests/` (PROBLEMATIC)

| File | Purpose | Issues | Action |
|------|---------|--------|--------|
| `quick_test.py` | Quick test? | Review - may not be pytest | Audit |
| `simple_structure_test.py` | Structure test? | Review | Audit |
| `test_structure_generation.py` | Structure gen test | Review | Audit |
| `run_tests.py` | Test runner script | Redundant with pytest | Delete |
| `__pycache__/` | Python cache | Should be gitignored | Delete |

**Issues:**
1. `run_tests.py` suggests not using pytest properly
2. Test naming inconsistent
3. `__pycache__` committed

**Action:**
- Audit tests for actual pytest compatibility
- Delete `run_tests.py` if redundant
- Standardize test naming: `test_*.py`

---

## Proposed Clean Structure

```
abmelt_infer_pipeline/
├── __init__.py                 # Package init with version
├── py.typed                    # PEP 561 marker
├── configs/
│   ├── paper_config.yaml
│   └── testing_config.yaml
├── mdp/                        # GROMACS params (keep as-is)
│   ├── em.mdp
│   ├── ions.mdp
│   ├── nvt.mdp
│   ├── npt.mdp
│   └── md.mdp
├── models/                     # Trained weights (keep as-is)
│   ├── tagg/
│   ├── tm/
│   └── tmon/
├── src/                        # Core modules
│   ├── __init__.py
│   ├── structure_prep.py
│   ├── preprocess.py
│   ├── md_simulation.py
│   ├── compute_descriptors.py
│   ├── model_inference.py
│   ├── order_param.py
│   └── res_sasa.py
├── tests/                      # Pytest tests
│   ├── __init__.py
│   ├── conftest.py             # Fixtures
│   ├── fixtures/               # Test data
│   │   └── example_antibody.pdb
│   ├── test_structure_prep.py
│   ├── test_preprocess.py
│   ├── test_descriptors.py
│   └── test_inference.py
├── cli.py                      # CLI entrypoint (renamed from infer.py)
└── Dockerfile
```

**Removed/Moved:**
- `envs/` → docs/how-to/ (guides only, delete pyproject.tomls)
- `demo_structure_generation.py` → examples/ or delete
- `infer_using_descriptors.py` → merge into cli.py or delete
- `feature_names.txt` → configs/ or models/
- `my_antibody.pdb` → tests/fixtures/
- `CLEANUP_GUIDE.md` → docs/how-to/
- `run_data/` → .gitignore (runtime output)
- All `__pycache__/` → delete

---

## Implementation Phases

### Phase 1: Audit (No Code Changes)
1. Read and document every file in `src/`
2. Compare each to `_AbMelt_reference/src/` equivalent
3. Compare to paper markdown for accuracy
4. Document discrepancies in `docs/bugs/`

### Phase 2: Cleanup (Safe Changes)
1. Delete `__pycache__/` directories
2. Delete `envs/` nested pyproject.toml and poetry.lock files
3. Move documentation files to proper locations
4. Add missing `__init__.py` files
5. Update .gitignore for run_data/

### Phase 3: Restructure (Directory Changes)
1. Move files to proper locations per proposed structure
2. Update imports throughout codebase
3. Update pyproject.toml paths

### Phase 4: Code Quality (Refactoring)
1. Add type hints
2. Fix linting issues
3. Standardize error handling
4. Add proper logging

### Phase 5: Testing (Validation)
1. Write/fix proper pytest tests
2. Validate against paper results
3. Add CI checks

---

## Comparison Points

### vs. `_AbMelt_reference/`

| Our File | Reference File | Match? |
|----------|---------------|--------|
| src/preprocess.py | src/preprocess.py | ? |
| src/order_param.py | src/order_param.py | ? |
| src/res_sasa.py | src/res_sasa.py | ? |
| models/*.pkl | models/*.pkl | ? (should be identical) |
| mdp/*.mdp | mdp/*.mdp | ? |

### vs. Paper (reference_literature/markdown/abmelt.md)

Key validation points:
1. MD simulation parameters (300K, 350K, 400K)
2. Descriptor calculations (Table 1 in paper)
3. Model architectures (KNN for Tagg, ElasticNet for Tm,on, RF for Tm)
4. Feature selection methodology

---

## Open Questions

1. Are the tests actual tests or demo scripts?
2. Is `infer_using_descriptors.py` needed or redundant?
3. What's the difference between `paper_config.yaml` and `testing_config.yaml`?
4. Are the pkl models binary-identical to reference?
5. Does the code actually run end-to-end?

---

## Next Steps

1. [ ] Deep audit of `src/` files - read every line
2. [ ] Compare to `_AbMelt_reference/src/`
3. [ ] Validate against paper methodology
4. [ ] Document findings in this spec
5. [ ] Execute phases 1-5 incrementally
