# Project Context & Intent - AbMelt Production Pipeline

## Origin & Collaboration

**From Hugging Science Discord - Georgia Channing (9/19/25, 8:46 AM):**

> This project is part of an HF collaboration with Ginkgo Datapoints. Building of these models and benchmarking of them will make you eligible to be an author on the resultant journal paper with Ginkgo.
>
> Antibody developability is the task of evaluating whether an antibody drug candidate's can become a manufacturable, stable, safe, and effective therapeutic. Assessing this is critical for good drug development.
>
> Right now, there are a bunch of models released by the big pharma companies and related labs, but they're a pain to build. What the community needs is to have a pre-built version of these models to use in larger pipelines and to benchmark the different methods on many datasets.
>
> The first models we need built and benchmarked are:
> AbMelt: https://github.com/MSDLLCpapers/AbMelt (code available, just needs build)

---

## What Problem Are We Solving?

### The Antibody Developability Challenge

**Antibody developability** = Can this antibody drug candidate become:
- Manufacturable at scale
- Stable during storage/shipping
- Safe for patients
- Effective as a therapeutic

**Why this matters:**
- Critical for drug development pipeline
- Failures downstream are EXPENSIVE
- Early prediction saves millions + years of work

### Current State of the Field

Big pharma companies (Merck, etc.) have published ML models to predict antibody stability:
- **AbMelt** (Merck) - Predicts thermal stability via MD simulations
- Other models from Genentech, AstraZeneca, etc.

**The Problem:**
1. Code is research-quality (hard to use)
2. Requires HPC clusters, GROMACS installation, manual setup
3. No standardized inference pipeline
4. No benchmarking across methods
5. Community can't easily compare models or integrate into pipelines

---

## What This Project Does

### High-Level Goal

**Productionize AbMelt** so the scientific community can:
1. **Use it easily** - Simple CLI, Docker containers, clear docs
2. **Benchmark it** - Compare against other developability models
3. **Integrate it** - Plug into larger antibody design/screening pipelines
4. **Publish results** - Contribute to HF/Ginkgo journal paper

### Specific Deliverables

1. **Working inference pipeline** - Take antibody sequence → predict stability
2. **Pre-trained models** - Load existing Merck models, ready to use
3. **Benchmarking framework** - Test on multiple datasets
4. **Clean codebase** - Tests, docs, type hints, CI/CD
5. **HuggingFace integration** - Models on HF Hub, datasets published

---

## Technical Overview

### What is AbMelt?

**Paper:** "AbMelt: Learning antibody thermostability from molecular dynamics" (Biophysical Journal, June 2024)

**Method:**
1. Take antibody sequence (heavy + light chains)
2. Generate 3D structure (ImmuneBuilder or experimental PDB)
3. Run molecular dynamics (MD) simulations at 3 temperatures (300K, 350K, 400K)
4. Compute structural descriptors (RMSD, RMSF, H-bonds, SASA, etc.)
5. Feed descriptors to trained Random Forest models
6. **Predict 3 stability metrics:**
   - **T_agg** - Aggregation temperature
   - **T_m,on** - Melting onset temperature
   - **T_m** - Melting temperature

**Original Implementation:**
- Research code from Merck (https://github.com/MSDLLCpapers/AbMelt)
- Requires GROMACS, MOE/ImmuneBuilder, manual PBS job submission
- Designed for HPC batch processing, not single-antibody inference
- **Pre-trained models ARE included** (`AbMelt/models/`)

---

## Repository Structure

### Current State (Praful's Work)

```
antibody-developability-abmelt/
├── AbMelt/                          # Original Merck research code (cloned as-is)
│   ├── models/                      # ✅ Pre-trained ML models (pkl files)
│   │   ├── tagg/efs_best_knn.pkl
│   │   ├── tm/efs_best_randomforest.pkl
│   │   └── tmon/efs_best_elasticnet.pkl
│   ├── src/                         # Original structure generation, MD, descriptors
│   ├── train_test_predictors/       # Training scripts & prediction.py
│   ├── data/                        # Training/test datasets
│   └── mdp/                         # GROMACS parameter files
│
└── abmelt_infer_pipeline/           # Praful's productionization wrapper
    ├── infer.py                     # Main CLI entry point
    ├── src/
    │   ├── structure_prep.py        # ✅ Structure generation wrapper
    │   ├── md_simulation.py         # ✅ MD simulation wrapper
    │   └── compute_descriptors.py   # ✅ Descriptor computation wrapper
    ├── configs/                     # YAML configs
    ├── tests/                       # Test scripts (not proper pytest)
    └── envs/                        # Conda environments
```

### What Works vs What Doesn't

**✅ WORKING:**
1. Structure generation from sequences (ImmuneBuilder)
2. Structure loading from PDB files
3. MD simulation pipeline (GROMACS integration)
4. Descriptor computation from trajectories
5. Skip flags for each pipeline step
6. Config management via YAML

**❌ MISSING (The Critical Gap):**
1. **ML model inference** - Step 4 of the pipeline (line 223 in `infer.py` says `# TODO: Step 4: ML prediction`)
2. The models exist at `AbMelt/models/`, just need to load and predict
3. Proper test suite (pytest, not manual scripts)
4. Type hints throughout
5. Linting/formatting setup
6. CI/CD pipeline
7. Documentation for users

---

## What We Need to Build

### Phase 1: Complete the Pipeline (CRITICAL)
**Status:** 80% done, missing final step

**Task:** Implement ML prediction step
- Load pre-trained models from `AbMelt/models/`
- Take descriptor DataFrame from `compute_descriptors()`
- Run predictions for Tagg, Tm,on, Tm
- Return results in structured format

**Files to create/modify:**
- `abmelt_infer_pipeline/src/prediction.py` (NEW)
- `abmelt_infer_pipeline/infer.py` (modify line 223)
- Tests for prediction step

**Estimated time:** 2-4 hours

**Reference:** `AbMelt/train_test_predictors/prediction.py` (original inference code)

---

### Phase 2: Production-Ready Code
**Status:** Not started

**Tasks:**
1. **Testing**
   - Proper pytest suite (not manual scripts)
   - Unit tests for each module
   - Integration tests for full pipeline
   - Test fixtures with small example antibodies
   - Coverage > 80%

2. **Type Safety**
   - Add type hints to all functions
   - MyPy strict mode passing
   - Pydantic models for configs/data validation

3. **Code Quality**
   - Ruff linting + formatting
   - Remove dead code
   - Fix hardcoded paths in `AbMelt/src/anaylze.py`
   - Clean up requirements (currently 424 lines of bloat)

4. **Developer Experience**
   - Makefile for common tasks
   - Pre-commit hooks
   - pyproject.toml with proper deps
   - CI/CD (GitHub Actions)

---

### Phase 3: Benchmarking & Validation
**Status:** Not started

**Tasks:**
1. **Dataset Collection**
   - AbMelt original datasets
   - Public antibody stability datasets (Jain et al., Shehata et al.)
   - Therapeutic antibodies with known stability data

2. **Benchmarking Framework**
   - Run predictions on all datasets
   - Compute metrics (MAE, R², correlation)
   - Compare against baselines
   - Statistical analysis

3. **Results Publishing**
   - Benchmark report
   - Figures and tables
   - Analysis notebooks

---

### Phase 4: Community Integration
**Status:** Not started

**Tasks:**
1. **HuggingFace Hub**
   - Upload models to HF Hub
   - Upload datasets
   - Create model cards with metadata

2. **Documentation**
   - User guide
   - API reference
   - Tutorial notebooks
   - Docker setup guide

3. **Packaging**
   - PyPI package
   - Docker images
   - Conda package (optional)

---

## GitHub Issues Prioritization

From Praful's repo (https://github.com/Praful932/antibody-developability-abmelt):

**Priority 1 (BLOCKERS):**
- **Issue #8: Test model inference** ← THE CRITICAL MISSING PIECE
  - Implement ML prediction step
  - This unblocks everything else

**Priority 2 (Important):**
- **Issue #3: Add inference env in dockerfile**
  - Easier deployment/testing
- **Issue #5: Publish datasets used in the paper to HF**
  - Enables benchmarking

**Priority 3 (Nice to have):**
- **Issue #7: Add block_length parameters in descriptor computation**
  - Scientific parameter tuning
- **Issue #4: Measure & report GPU benchmark times**
  - Performance metrics

---

## Current Analysis (Our Findings)

### Code Quality Issues Found

1. **`AbMelt/src/anaylze.py`**
   - Filename typo: `anaylze.py` → should be `analyze.py`
   - Hardcoded placeholder paths that will never work:
     ```python
     os.chdir('pathway/to/executable/.py')  # Line 32
     os.chdir('pathway/to/data/AbMelt')     # Line 37
     ```
   - Magic numbers (t0 = 2000) hardcoded everywhere
   - This file is **unusable in current state**

2. **`AbMelt/requirements.txt`**
   - 424 lines (BLOATED)
   - Duplicate packages (pandas 3x, numpy multiple times)
   - Mixing GPU-specific deps (CUDA libraries) with general deps
   - No clear dev vs prod separation
   - **Needs complete rebuild**

3. **`abmelt_infer_pipeline/tests/`**
   - Not real unit tests
   - Just runner scripts, no assertions
   - No pytest framework
   - **Needs rewrite from scratch**

4. **Type Hints**
   - Almost none throughout codebase
   - Can't run MyPy
   - Poor IDE support

5. **Documentation**
   - `README.md` is 3 lines: "# antibody-developability-abmelt\nabmelt benchmark"
   - No user guide
   - No API docs

### Praful's Approach (His Wrapper Strategy)

**Good decisions:**
- Kept original `AbMelt/` code untouched (clean separation)
- Built wrapper in `abmelt_infer_pipeline/` with modern patterns
- Uses `sys.path.append` to import original code (non-invasive)
- Added skip flags for debugging pipeline steps
- YAML config management

**Incomplete work:**
- 80% done, missing final ML prediction step
- No proper tests
- No type hints
- No linting/formatting
- No CI/CD

**Our assessment:**
- His architecture is solid
- Just needs the last 20% + production hardening
- **Could finish core functionality in 1 day** (as suspected)
- **Production-ready in 1 week** with proper tooling

---

## Success Criteria

### Minimum Viable (MVP)
- [ ] ML prediction step implemented
- [ ] Full pipeline works end-to-end (sequence → predictions)
- [ ] Can run on example antibody (alemtuzumab)
- [ ] Basic tests pass

### Production Ready
- [ ] Pytest suite with >80% coverage
- [ ] Type hints + MyPy passing
- [ ] Ruff linting/formatting
- [ ] CI/CD pipeline
- [ ] Docker container
- [ ] User documentation

### Research Ready
- [ ] Benchmarking on multiple datasets
- [ ] Statistical validation
- [ ] Comparison with baselines
- [ ] Results published
- [ ] Models on HuggingFace Hub

### Journal Paper Ready
- [ ] All above complete
- [ ] Reproducible experiments
- [ ] Methods section written
- [ ] Figures and tables
- [ ] Co-authorship with Ginkgo confirmed

---

## Timeline Estimate

**Week 1: Core Pipeline**
- Day 1-2: Implement ML prediction step
- Day 3-4: Proper test suite
- Day 5: Type hints + linting
- Day 6-7: Documentation

**Week 2: Production Hardening**
- Day 1-2: CI/CD setup
- Day 3-4: Docker + deployment
- Day 5-7: Bug fixes + optimization

**Week 3: Benchmarking**
- Day 1-2: Dataset collection
- Day 3-5: Run benchmarks
- Day 6-7: Analysis + report

**Week 4: Publication**
- Day 1-3: HuggingFace Hub upload
- Day 4-5: Final documentation
- Day 6-7: Paper submission prep

**Total: ~1 month of focused work**

---

## Key Contacts & Resources

**HuggingFace Collaboration:**
- Georgia Channing (Hugging Science Discord)
- Ginkgo Datapoints team

**Original Authors:**
- Zachary A. Rollins (Merck)
- Paper: https://www.cell.com/biophysj/abstract/S0006-3495(24)00385-0
- Code: https://github.com/MSDLLCpapers/AbMelt
- Data: https://zenodo.org/records/10815667

**Current Maintainer:**
- Praful932: https://github.com/Praful932/antibody-developability-abmelt

---

## For AI Agents Working on This

### If you're implementing ML prediction:
1. Look at `AbMelt/train_test_predictors/prediction.py` (reference implementation)
2. Models are at `AbMelt/models/{tagg,tm,tmon}/efs_best_*.pkl`
3. Use joblib to load pkl files
4. Input: DataFrame from `compute_descriptors()`
5. Output: Dict with `{"tagg": float, "tm_on": float, "tm": float}`

### If you're writing tests:
1. See `burner_docs/MODERN_DEVEX_SETUP.md` for pytest setup
2. Use small example antibodies (alemtuzumab is in repo)
3. Test each module independently
4. Use fixtures for common data

### If you're setting up DevEx:
1. Follow `burner_docs/MODERN_DEVEX_SETUP.md` exactly
2. Use uv for package management
3. Use Ruff for linting/formatting
4. Use MyPy for type checking
5. Makefile for automation

### If you're confused:
1. Read this doc first
2. Check `MODERN_DEVEX_SETUP.md` for tooling
3. Look at original AbMelt README: `AbMelt/README.md`
4. Ask for clarification before making assumptions

---

## Why This Matters

**Scientific Impact:**
- Enable antibody drug discovery at scale
- Reduce development costs/time
- Improve success rates for therapeutics

**Open Science:**
- Make pharma models accessible to everyone
- Reproducible research
- Community benchmarking

**Personal Impact:**
- Co-authorship on Ginkgo journal paper
- Contribution to high-impact bioinformatics
- Skills in ML + computational biology

**Let's build this right. Let's build this together.**

---

*Last updated: 2025-11-24*
*Context for AI agents tag-teaming this project*
