# SPEC-003: Cleanup Plan - Remove Cruft from abmelt_infer_pipeline

**Status:** Draft
**Created:** 2024-12-01
**Depends On:** SPEC-002 (Pipeline Audit)
**Leads To:** SPEC-004 (Wrapper-to-Implementation Mapping)

## Purpose

Clean up `abmelt_infer_pipeline/` to remove cruft while preserving files that serve as a **guide** for what we need to implement. After this cleanup, the directory becomes a clean reference showing exactly what functionality the wrapper was trying to provide.

## Proposed Rename

```
abmelt_infer_pipeline/ → _abmelt_wrapper_reference/
```

**Rationale:**
- Underscore prefix indicates "reference only, not production code"
- Matches naming convention of `_AbMelt_reference/`
- Makes clear this is not the implementation we'll ship

---

## Cleanup Categories

### 1. DELETE - Pure Cruft (No Value)

These files provide no guidance and should be removed:

| File/Directory | Reason |
|----------------|--------|
| `run_data/` | Runtime output, shouldn't be in repo |
| `src/__pycache__/` | Python cache (if exists) |
| `tests/__pycache__/` | Python cache (if exists) |
| `envs/abmelt_md_simulation_env/pyproject.toml` | Duplicate config, violates SSOT |
| `envs/abmelt_md_simulation_env/poetry.lock` | Stale lock file |
| `envs/abmelt_sklearn_model_infer_env/pyproject.toml` | Duplicate config, violates SSOT |
| `envs/abmelt_sklearn_model_infer_env/poetry.lock` | Stale lock file |
| `tests/run_tests.py` | Redundant test runner |
| `my_antibody.pdb` | Example file, not needed in reference |

### 2. MOVE - Useful but Wrong Location

These files have value but belong elsewhere:

| File | Current Location | Move To | Reason |
|------|------------------|---------|--------|
| `CLEANUP_GUIDE.md` | root | `docs/how-to/cleanup-temp-files.md` | Documentation |
| `envs/*/linux_cuda_machine_setup.md` | envs/ | `docs/how-to/` | Setup guides |
| `envs/*/mac_env_setup.md` | envs/ | `docs/how-to/` | Setup guides |
| `envs/*/*.sh` | envs/ | `scripts/setup/` | Shell scripts |
| `src/REQUIRED_FILES.md` | src/ | `docs/reference/` | Documentation |
| `feature_names.txt` | root | `docs/reference/` or `models/` | Feature reference |

### 3. KEEP AS REFERENCE - Guide for Implementation

These files show **what** needs to be implemented. They stay in the renamed `_abmelt_wrapper_reference/`:

| File | Purpose as Guide |
|------|------------------|
| `infer.py` | Shows the 4-step pipeline orchestration pattern |
| `src/structure_prep.py` | Shows structure preparation flow (ImmuneBuilder, chain renaming) |
| `src/preprocess.py` | Shows GROMACS preprocessing (PropKa, CDR indexing) |
| `src/md_simulation.py` | Shows MD workflow (setup, equilibration, production) |
| `src/compute_descriptors.py` | Shows which GROMACS analyses to run, feature naming |
| `src/model_inference.py` | Shows exact features needed per model |
| `src/order_param.py` | Shows S² calculation algorithm (with bugs to fix) |
| `src/res_sasa.py` | Shows core/surface SASA algorithm (with bugs to fix) |
| `src/cleanup_temp_files.py` | Shows which files are intermediate vs required |
| `configs/*.yaml` | Shows configuration schema and default values |
| `mdp/*.mdp` | GROMACS parameter files (keep as-is) |
| `models/` | Trained model weights (keep as-is, may move to root) |
| `Dockerfile` | Shows containerization approach |

### 4. REVIEW - May Have Value

These need closer inspection:

| File | Question |
|------|----------|
| `demo_structure_generation.py` | Keep as example or delete? |
| `infer_using_descriptors.py` | Useful for testing inference-only? |
| `tests/quick_test.py` | Worth converting to pytest? |
| `tests/simple_structure_test.py` | Worth converting to pytest? |
| `tests/test_structure_generation.py` | Worth converting to pytest? |
| `mdp/arr.sh` | HPC-specific, needed? |

---

## Cleanup Execution Plan

### Step 1: Create Backup Branch
```bash
git checkout -b backup/abmelt-wrapper-original
git checkout feat/devex-tooling
```

### Step 2: Delete Pure Cruft
```bash
# Runtime output
rm -rf abmelt_infer_pipeline/run_data/

# Pycache (if exists)
find abmelt_infer_pipeline/ -type d -name "__pycache__" -exec rm -rf {} +

# Duplicate configs
rm abmelt_infer_pipeline/envs/abmelt_md_simulation_env/pyproject.toml
rm abmelt_infer_pipeline/envs/abmelt_md_simulation_env/poetry.lock
rm abmelt_infer_pipeline/envs/abmelt_sklearn_model_infer_env/pyproject.toml
rm abmelt_infer_pipeline/envs/abmelt_sklearn_model_infer_env/poetry.lock

# Redundant files
rm abmelt_infer_pipeline/tests/run_tests.py
rm abmelt_infer_pipeline/my_antibody.pdb
```

### Step 3: Move Documentation
```bash
# Create target directories
mkdir -p docs/how-to/environment-setup
mkdir -p docs/reference
mkdir -p scripts/setup

# Move documentation
mv abmelt_infer_pipeline/CLEANUP_GUIDE.md docs/how-to/cleanup-temp-files.md
mv abmelt_infer_pipeline/src/REQUIRED_FILES.md docs/reference/required-files.md
mv abmelt_infer_pipeline/feature_names.txt docs/reference/feature-names.txt

# Move setup guides
mv abmelt_infer_pipeline/envs/abmelt_md_simulation_env/linux_cuda_machine_setup.md docs/how-to/environment-setup/
mv abmelt_infer_pipeline/envs/abmelt_md_simulation_env/mac_env_setup.md docs/how-to/environment-setup/md-simulation-mac.md
mv abmelt_infer_pipeline/envs/abmelt_sklearn_model_infer_env/mac_env_setup.md docs/how-to/environment-setup/inference-mac.md

# Move shell scripts
mv abmelt_infer_pipeline/envs/abmelt_md_simulation_env/linux_cuda_machine_setup_1.sh scripts/setup/
mv abmelt_infer_pipeline/envs/abmelt_md_simulation_env/linux_cuda_machine_setup_2.sh scripts/setup/

# Remove now-empty envs READMEs
rm abmelt_infer_pipeline/envs/abmelt_md_simulation_env/README.md
rm -rf abmelt_infer_pipeline/envs/  # Should be empty now
```

### Step 4: Rename Directory
```bash
mv abmelt_infer_pipeline/ _abmelt_wrapper_reference/
```

### Step 5: Update pyproject.toml References
```toml
# Update paths in pyproject.toml
[tool.ruff]
extend-exclude = [
    "_AbMelt_reference/**",
    "_abmelt_wrapper_reference/**",  # NEW
    "reference_repos/**",
    "burner_docs/**",
]

[tool.mypy]
exclude = [
    "_AbMelt_reference/.*",
    "_abmelt_wrapper_reference/.*",  # NEW
    "reference_repos/.*",
    "burner_docs/.*",
]
```

### Step 6: Update .gitignore
Add to root `.gitignore`:
```
# Reference directories (kept for guidance only)
_AbMelt_reference/
_abmelt_wrapper_reference/

# But explicitly track what we need
!_abmelt_wrapper_reference/configs/
!_abmelt_wrapper_reference/mdp/
!_abmelt_wrapper_reference/models/
```

---

## Post-Cleanup Directory Structure

```
_abmelt_wrapper_reference/
├── configs/
│   ├── paper_config.yaml
│   └── testing_config.yaml
├── mdp/
│   ├── em.mdp
│   ├── ions.mdp
│   ├── md.mdp
│   ├── npt.mdp
│   └── nvt.mdp
├── models/
│   ├── README.md
│   ├── tagg/
│   ├── tm/
│   └── tmon/
├── src/
│   ├── cleanup_temp_files.py
│   ├── compute_descriptors.py
│   ├── md_simulation.py
│   ├── model_inference.py
│   ├── order_param.py         # ⚠️ BUG: duplicate loop
│   ├── preprocess.py
│   ├── res_sasa.py            # ⚠️ BUG: inverted slice
│   └── structure_prep.py
├── tests/
│   ├── quick_test.py
│   ├── simple_structure_test.py
│   └── test_structure_generation.py
├── .gitignore
├── demo_structure_generation.py
├── Dockerfile
├── infer_using_descriptors.py
├── infer.py
└── README.md
```

---

## What This Enables

After cleanup, we have:

1. **Clean reference** showing what Praful's wrapper was trying to do
2. **Documentation moved** to proper Diátaxis structure
3. **No duplicate configs** - single source of truth in root
4. **Clear delineation** - underscore prefix = reference only

This sets up SPEC-004 to create a mapping:
- `_abmelt_wrapper_reference/src/X.py` → Points to `_AbMelt_reference/src/X.py` (original)
- Shows what needs to be extracted and properly implemented

---

## Spec Roadmap

| Spec | Purpose |
|------|---------|
| SPEC-001 | Original refactoring plan (overview) |
| SPEC-002 | Deep audit of current state (complete) |
| **SPEC-003** | **Cleanup cruft, rename to reference** |
| SPEC-004 | Wrapper → Original mapping (what wraps what) |
| SPEC-005 | Extract & implement properly (new `abmelt/` package) |
| SPEC-006 | Final refactoring & polish |

---

## Checklist

- [ ] Create backup branch
- [ ] Delete pure cruft (run_data, pycache, duplicate configs)
- [ ] Move documentation to docs/
- [ ] Move scripts to scripts/
- [ ] Remove empty envs/ directory
- [ ] Rename directory to `_abmelt_wrapper_reference/`
- [ ] Update pyproject.toml excludes
- [ ] Update .gitignore
- [ ] Commit with clear message
