# SPEC-000: Meta Map - Project Overview & Spec Roadmap

**Status:** Living Document
**Created:** 2024-12-01
**Last Updated:** 2024-12-01

---

## Project Identity

**Name:** AbMelt Inference Pipeline
**Purpose:** Predict antibody thermostability (T_agg, T_m, T_m_onset) from sequence or structure using MD simulation-derived features and trained ML models.
**Origin:** Based on the AbMelt paper by Merck (MIT License)

---

## Directory Architecture

```
antibody-developability-abmelt/
│
├── docs/                           # Diátaxis documentation
│   ├── specs/                      # Implementation specifications
│   │   ├── 000-meta-map.md         # ← YOU ARE HERE
│   │   ├── 001-pipeline-refactor-plan.md
│   │   ├── 002-pipeline-audit-current-state.md
│   │   └── 003-cleanup-plan.md
│   ├── tutorials/                  # Learning-oriented
│   ├── how-to/                     # Task-oriented
│   ├── explanation/                # Understanding-oriented
│   ├── reference/                  # Information-oriented
│   ├── adr/                        # Architecture Decision Records
│   └── bugs/                       # Known issues
│
├── _AbMelt_reference/              # Original AbMelt repo (READ-ONLY reference)
│   ├── src/                        # Original implementation
│   ├── models/                     # Trained model weights
│   └── ...
│
├── abmelt_infer_pipeline/          # Praful's wrapper (TO BE CLEANED → _abmelt_wrapper_reference/)
│   ├── src/                        # Wrapper code with bugs
│   ├── configs/                    # YAML configs
│   ├── mdp/                        # GROMACS params
│   ├── models/                     # Model weights (copy)
│   └── ...
│
├── abmelt/                         # FUTURE: Clean implementation (SPEC-005)
│   ├── __init__.py
│   ├── pipeline/
│   ├── models/
│   └── ...
│
├── pyproject.toml                  # Single source of truth for dependencies
├── uv.lock                         # Locked dependencies
└── .github/workflows/              # CI/CD
```

---

## Spec Roadmap

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SPEC DEPENDENCY GRAPH                          │
└─────────────────────────────────────────────────────────────────────────────┘

  SPEC-001: Pipeline Refactor Plan (Overview)
      │
      ▼
  SPEC-002: Deep Audit ────────────────────────────────────┐
      │                                                     │
      │  Documents:                                         │
      │  • All files in abmelt_infer_pipeline/              │
      │  • Critical bugs (SASA slice, S² duplicate)         │
      │  • Broken imports                                   │
      │                                                     │
      ▼                                                     │
  SPEC-003: Cleanup Plan                                    │
      │                                                     │
      │  Actions:                                           │
      │  • Delete cruft (run_data, duplicate configs)       │
      │  • Move docs to docs/                               │
      │  • Rename to _abmelt_wrapper_reference/             │
      │                                                     │
      ▼                                                     │
  SPEC-004: Wrapper → Original Mapping ◄───────────────────┘
      │
      │  Shows:
      │  • Which wrapper file wraps which original file
      │  • What sys.path.append was trying to import
      │  • Gap analysis (what wrapper adds vs original)
      │
      ▼
  SPEC-005: Extract & Implement
      │
      │  Actions:
      │  • Create new abmelt/ package
      │  • Extract algorithms from _AbMelt_reference/
      │  • Fix known bugs during extraction
      │  • Proper Python package structure
      │
      ▼
  SPEC-006: Final Polish
      │
      │  Actions:
      │  • Add type hints throughout
      │  • Write pytest tests
      │  • Documentation
      │  • CI/CD validation
      │
      ▼
  [PRODUCTION READY]
```

---

## Spec Status Matrix

| Spec | Title | Status | Key Deliverable |
|------|-------|--------|-----------------|
| 000 | Meta Map | Living | This document |
| 001 | Pipeline Refactor Plan | Draft | High-level plan, phases |
| 002 | Pipeline Audit | Complete | File-by-file documentation, bugs identified |
| 003 | Cleanup Plan | Draft | Cruft removal checklist, rename plan |
| 004 | Wrapper Mapping | Planned | Wrapper→Original file mapping |
| 005 | Extract & Implement | Planned | New `abmelt/` package |
| 006 | Final Polish | Planned | Production-ready code |

---

## Key Reference Materials

### Code References
| Location | Purpose | Status |
|----------|---------|--------|
| `_AbMelt_reference/` | Original Merck implementation | READ-ONLY |
| `abmelt_infer_pipeline/` | Praful's wrapper | TO BE CLEANED |
| `abmelt/` | Clean implementation | NOT YET CREATED |

### Documentation References
| Location | Purpose |
|----------|---------|
| `reference_literature/markdown/abmelt.md` | Paper methodology (local exclude) |
| SPEC-002 | Current state documentation |

### Model Weights
| Model | Algorithm | Location |
|-------|-----------|----------|
| T_agg | KNN | `*/models/tagg/efs_best_knn.pkl` |
| T_m | Random Forest | `*/models/tm/efs_best_randomforest.pkl` |
| T_m_onset | ElasticNet | `*/models/tmon/efs_best_elasticnet.pkl` |

---

## Critical Bugs (from SPEC-002)

| Bug | File | Severity | Fix Spec |
|-----|------|----------|----------|
| SASA slice inverted | `res_sasa.py:47-48` | CRITICAL | SPEC-005 |
| Duplicate S² loop | `order_param.py:261-310` | HIGH | SPEC-005 |
| Broken imports | `structure_prep.py:14`, `md_simulation.py:15` | BLOCKING | SPEC-003/005 |

---

## Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ABMELT PIPELINE FLOW                             │
└─────────────────────────────────────────────────────────────────────────────┘

  INPUT                     STEP 1                    STEP 2
  ─────                     ──────                    ──────
  Sequences (H/L)  ───┐     Structure Prep            MD Simulation
       OR             ├──►  • ImmuneBuilder      ──►  • GROMACS
  PDB file        ───┘     • Chain rename (H/L→A/B)   • 3 temps (300K, 350K, 400K)
                           • Validation               • 100ns production

                           STEP 3                    STEP 4
                           ──────                    ──────
                           Descriptor Computation    Model Inference
                      ──►  • GROMACS analysis   ──►  • Load sklearn models
                           • S² order params         • Extract features
                           • Core/surface SASA       • Predict T_agg, T_m, T_m_onset
                           • Lambda slopes

  OUTPUT
  ──────
  predictions.csv
  • T_agg (°C)
  • T_m (°C)
  • T_m_onset (°C)
```

---

## Feature Requirements (Quick Reference)

| Feature | Model | Temperature | Source |
|---------|-------|-------------|--------|
| `rmsf_cdrs_mu_400` | T_agg | 400K | GROMACS rmsf |
| `gyr_cdrs_Rg_std_400` | T_agg | 400K | GROMACS gyrate |
| `all-temp_lamda_b=25_eq=20` | T_agg | All | S² regression |
| `gyr_cdrs_Rg_std_350` | T_m | 350K | GROMACS gyrate |
| `bonds_contacts_std_350` | T_m, T_m_onset | 350K | GROMACS hbond |
| `rmsf_cdrl1_std_350` | T_m | 350K | GROMACS rmsf |
| `all-temp-sasa_core_mean_k=20_eq=20` | T_m_onset | All | mdtraj SASA |
| `all-temp-sasa_core_std_k=20_eq=20` | T_m_onset | All | mdtraj SASA |
| `r-lamda_b=2.5_eq=20` | T_m_onset | All | S² regression R² |

---

## External Dependencies

### Python (via uv/pyproject.toml)
- `torch>=2.0.0`
- `immunebuilder>=1.2`
- `gromacswrapper>=0.9.2`
- `mdanalysis>=2.6.0`
- `scikit-learn==1.2.0` (pinned!)
- `mdtraj>=1.9.9`
- `biopython>=1.79`

### System
- GROMACS 2024 (with CUDA for GPU)
- CHARMM force field

### Conda (not in pyproject.toml)
- `openmm`
- `pdbfixer`
- `anarci`

---

## Next Actions

1. **Execute SPEC-003** - Clean up cruft, rename directory
2. **Write SPEC-004** - Map wrapper to original
3. **Write SPEC-005** - Extraction & implementation plan
4. **Execute SPEC-005** - Build clean `abmelt/` package
5. **Write/Execute SPEC-006** - Final polish

---

## Changelog

| Date | Change |
|------|--------|
| 2024-12-01 | Initial meta map created |
| 2024-12-01 | Added specs 001-003 |
