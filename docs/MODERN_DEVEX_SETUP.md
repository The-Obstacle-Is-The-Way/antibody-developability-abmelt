# Modern Python ML DevEx Setup - November 2025

## What This Repo Is Missing (And Why We Can't Work Like This)

Current state: **No test suite, no linters, no formatters, no type checking = SLOP**

This is an AI-native development environment. We need proper tooling to stay on track and push quality PRs.

---

## The 2025 Modern Python Stack

### Core Philosophy
- **Speed**: Tools written in Rust (uv, ruff) are 10-100x faster
- **Simplicity**: One tool per job, minimal config
- **Standards**: Follow PEPs, use modern best practices
- **AI-First**: Type hints, clear tests, automated checks

---

## Required Tooling

### 1. **uv** - Package Manager & Environment
**What**: Replaces pip, poetry, virtualenv, pipx, pyenv - ALL IN ONE
**Why**: Written in Rust, insanely fast, handles everything

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create project
uv init

# Add dependencies
uv add pytest ruff mypy

# Run commands
uv run pytest
uv run python script.py
```

**Key Features**:
- Auto-manages virtual environments (no more manual venv)
- Cross-platform lockfiles
- Global cache for speed
- Standards-compliant (follows PEPs)

**Sources**:
- [uv: Unified Python packaging](https://astral.sh/blog/uv-unified-python-packaging)
- [Poetry vs UV in 2025](https://medium.com/@hitorunajp/poetry-vs-uv-which-python-package-manager-should-you-use-in-2025-4212cb5e0a14)

---

### 2. **Ruff** - Linting & Formatting
**What**: Replaces black, flake8, isort - ALL IN ONE
**Why**: 10-100x faster, auto-fixes issues, one config

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
]
ignore = ["E501"]  # line too long (let formatter handle it)

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

```bash
# Check
uv run ruff check .

# Auto-fix
uv run ruff check . --fix

# Format
uv run ruff format .
```

**Sources**:
- [A Modern Python Toolkit: Pydantic, Ruff, MyPy, and UV](https://dev.to/devasservice/a-modern-python-toolkit-pydantic-ruff-mypy-and-uv-4b2f)

---

### 3. **MyPy** - Static Type Checking
**What**: Catches type errors before runtime
**Why**: Better code quality, better IDE support, prevents bugs

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
strict_equality = true

# Per-module options
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

```bash
# Check types
uv run mypy abmelt_infer_pipeline/src/
```

**Note**: MyPy can be slow on large projects. Astral is working on a Rust-based alternative.

**Sources**:
- [Modern Python Project Setup 2025](https://albertsikkema.com/python/development/best-practices/2025/10/31/modern-python-project-setup.html)

---

### 4. **Pytest** - Testing Framework
**What**: The standard Python testing framework
**Why**: Simple, powerful, great plugin ecosystem

**Essential Plugins**:
- **pytest-cov** (87.7M downloads) - Coverage reports
- **pytest-xdist** (60.3M downloads) - Parallel testing across CPUs
- **pytest-sugar** - Beautiful test output with progress bars
- **pytest-asyncio** - Async test support
- **pytest-mock** - Easy mocking
- **pytest-timeout** - Kill hanging tests

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "-v",                    # verbose
    "--strict-markers",      # error on unknown markers
    "--tb=short",           # short traceback format
    "--cov=src",            # coverage for src/
    "--cov-report=term-missing",  # show missing lines
    "--cov-report=html",    # HTML coverage report
    "-n=auto",              # parallel testing (pytest-xdist)
    "--timeout=300",        # 5min timeout per test
]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
```

```bash
# Run tests
uv run pytest

# Run specific tests
uv run pytest tests/test_structure.py -k "test_pdb"

# Run in parallel
uv run pytest -n 4

# Skip slow tests
uv run pytest -m "not slow"

# Coverage report
uv run pytest --cov-report=html
open htmlcov/index.html
```

**Sources**:
- [8 Useful Pytest Plugins](https://pytest-with-eric.com/pytest-best-practices/pytest-plugins/)
- [Top pytest Plugins](https://pythontest.com/top-pytest-plugins/)

---

### 5. **Makefile** - Task Automation
**What**: Standardized commands for common tasks
**Why**: One command to rule them all, works everywhere

```makefile
# Makefile
.PHONY: help install test lint format typecheck clean all

# Default target
.DEFAULT_GOAL := help

# Variables
PYTHON := uv run python
PYTEST := uv run pytest
RUFF := uv run ruff
MYPY := uv run mypy

help:  ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

install:  ## Install dependencies
	uv sync

test:  ## Run tests with coverage
	$(PYTEST) -v --cov=abmelt_infer_pipeline/src --cov-report=term-missing --cov-report=html

test-fast:  ## Run tests in parallel
	$(PYTEST) -n auto

lint:  ## Run linting checks
	$(RUFF) check .

lint-fix:  ## Run linting with auto-fix
	$(RUFF) check . --fix

format:  ## Format code
	$(RUFF) format .

format-check:  ## Check if code is formatted
	$(RUFF) format --check .

typecheck:  ## Run type checking
	$(MYPY) abmelt_infer_pipeline/src/

clean:  ## Clean up cache and build artifacts
	find . -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name ".pytest_cache" -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name ".ruff_cache" -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name ".mypy_cache" -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name "htmlcov" -exec rm -rf {} + 2>/dev/null || true
	find . -type f -name ".coverage" -delete 2>/dev/null || true
	find . -type f -name "*.pyc" -delete 2>/dev/null || true

all: lint typecheck test  ## Run all checks (lint, typecheck, test)

ci: format-check lint typecheck test  ## Run all CI checks
```

**Usage**:
```bash
make                # Show help
make install        # Install deps
make test           # Run tests
make lint-fix       # Fix lint issues
make format         # Format code
make all            # Run everything
```

**Sources**:
- [Advanced Makefile Tips for Python Projects](https://glinteco.com/en/post/advanced-makefile-tips-tricks-and-best-practices-for-python-projects/)
- [Creating a Python Makefile](https://earthly.dev/blog/python-makefile/)

---

### 6. **Pre-commit** - Git Hooks
**What**: Automatic checks before committing
**Why**: Catch issues before they hit the repo

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.7.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-merge-conflict
```

```bash
# Install hooks
uv run pre-commit install

# Run manually
uv run pre-commit run --all-files
```

---

## Project Structure

```
antibody-developability-abmelt/
├── AbMelt/                       # Vendored MSD research code (do NOT lint/type/test)
├── abmelt_infer_pipeline/        # Wrapper we own
│   ├── src/                      # First-party code
│   ├── tests/                    # Tests mirror src/
│   ├── configs/                  # YAML config samples
│   ├── envs/                     # Conda/uv env definitions
│   ├── mdp/                      # MD parameter files
│   └── run_data/                 # Generated artifacts
├── burner_docs/                  # Local notes (git-excluded)
├── pyproject.toml               # Project config
├── Makefile                     # Task automation
├── .pre-commit-config.yaml      # Git hooks
└── README.md                    # User docs
```

---

## Scope: linting / typing / testing for this repo

- Treat `AbMelt/` (and anything in `reference_repos/`) as vendored research code: runtime imports allowed, but **no linting, formatting, typing, or coverage**.
- Enforce ruff/mypy/pytest/coverage only on our wrapper code under `abmelt_infer_pipeline/src` and its tests in `abmelt_infer_pipeline/tests`.
- Keep `burner_docs/` out of all tooling noise.
- Config hints for this repo:
  - Ruff: `extend-exclude = ["AbMelt/**", "reference_repos/**", "burner_docs/**"]`
  - MyPy: `exclude = ["AbMelt/", "reference_repos/", "burner_docs/"]` plus `[[tool.mypy.overrides]] module = ["AbMelt.*"] ignore_errors = true`
  - Coverage: `omit = ["AbMelt/*", "reference_repos/*", "burner_docs/*", "abmelt_infer_pipeline/tests/*"]`
- If we vendor more upstream code later, treat it the same way: never force our style/types/tests onto upstream drops.

---

## pyproject.toml - Complete Example

```toml
[project]
name = "abmelt-infer"
version = "0.1.0"
description = "Production-ready inference pipeline for AbMelt antibody thermostability prediction"
authors = [
    {name = "Your Name", email = "your.email@example.com"}
]
requires-python = ">=3.11"
dependencies = [
    "biopython>=1.81",
    "numpy>=1.26.0",
    "pandas>=2.1.0",
    "scikit-learn>=1.3.0",
    "joblib>=1.3.0",
    "pyyaml>=6.0",
    "mdanalysis>=2.6.0",
    "mdtraj>=1.9.9",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-cov>=4.1.0",
    "pytest-xdist>=3.5.0",
    "pytest-sugar>=1.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-mock>=3.12.0",
    "pytest-timeout>=2.2.0",
    "ruff>=0.7.0",
    "mypy>=1.13.0",
    "pre-commit>=3.6.0",
    "types-PyYAML",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# Ruff configuration
[tool.ruff]
line-length = 100
target-version = "py311"
src = ["abmelt_infer_pipeline/src", "abmelt_infer_pipeline/tests"]
extend-exclude = ["AbMelt/**", "reference_repos/**", "burner_docs/**"]

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "ARG", # flake8-unused-arguments
    "SIM", # flake8-simplify
]
ignore = [
    "E501",  # line too long (formatter handles it)
]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
docstring-code-format = true

# MyPy configuration
[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
strict_equality = true
exclude = [
    "AbMelt/",
    "reference_repos/",
    "burner_docs/",
]

[[tool.mypy.overrides]]
module = "abmelt_infer_pipeline.tests.*"
disallow_untyped_defs = false

[[tool.mypy.overrides]]
module = ["AbMelt.*"]
ignore_errors = true

# Pytest configuration
[tool.pytest.ini_options]
testpaths = ["abmelt_infer_pipeline/tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "-v",
    "--strict-markers",
    "--tb=short",
    "--cov=abmelt_infer_pipeline/src",
    "--cov-report=term-missing",
    "--cov-report=html",
    "-n=auto",
    "--timeout=300",
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]

# Coverage configuration
[tool.coverage.run]
source = ["abmelt_infer_pipeline/src"]
omit = [
    "abmelt_infer_pipeline/tests/*",
    "*/test_*.py",
    "AbMelt/*",
    "reference_repos/*",
    "burner_docs/*",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
]
```

---

## Quick Start Checklist

```bash
# 1. Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Create pyproject.toml with config above

# 3. Create Makefile with targets above

# 4. Install dependencies
make install

# 5. Set up pre-commit
uv add --dev pre-commit
uv run pre-commit install

# 6. Create .pre-commit-config.yaml

# 7. Run initial checks
make format
make lint-fix
make typecheck
make test

# 8. Commit
git add .
git commit -m "Add modern DevEx tooling"
```

---

## Why This Matters for AI-Native Development

1. **Type Hints** → Claude understands your code better
2. **Tests** → Validate changes without breaking things
3. **Linting** → Catch issues immediately
4. **Formatting** → Consistent code style
5. **Fast Tools** → Iterate quickly, stay in flow
6. **Automation** → `make all` before every PR

**Without these tools, we're coding blind. With them, we're unstoppable.**

---

## Sources

- [Modern Python Project Setup 2025](https://albertsikkema.com/python/development/best-practices/2025/10/31/modern-python-project-setup.html)
- [A Modern Python Toolkit: Pydantic, Ruff, MyPy, and UV](https://dev.to/devasservice/a-modern-python-toolkit-pydantic-ruff-mypy-and-uv-4b2f)
- [Poetry vs UV in 2025](https://medium.com/@hitorunajp/poetry-vs-uv-which-python-package-manager-should-you-use-in-2025-4212cb5e0a14)
- [uv: Unified Python packaging](https://astral.sh/blog/uv-unified-python-packaging)
- [8 Useful Pytest Plugins](https://pytest-with-eric.com/pytest-best-practices/pytest-plugins/)
- [Top pytest Plugins](https://pythontest.com/top-pytest-plugins/)
- [Advanced Makefile Tips for Python Projects](https://glinteco.com/en/post/advanced-makefile-tips-tricks-and-best-practices-for-python-projects/)
- [Creating a Python Makefile](https://earthly.dev/blog/python-makefile/)
