# Documentation Structure (Diátaxis Framework)

This documentation follows the [Diátaxis](https://diataxis.fr/) framework.

## Quadrants

| | **Learning** | **Working** |
|---|---|---|
| **Practical** | [tutorials/](tutorials/) | [how-to/](how-to/) |
| **Theoretical** | [explanation/](explanation/) | [reference/](reference/) |

### tutorials/
*Learning-oriented* - Get started with the project step-by-step.
- Aimed at newcomers
- "Follow along and learn"

### how-to/
*Task-oriented* - Solve specific problems.
- Assumes basic competence
- "How do I do X?"

### reference/
*Information-oriented* - Technical descriptions.
- Dry, accurate, complete
- API docs, config options, file formats

### explanation/
*Understanding-oriented* - Background and context.
- Why things are the way they are
- Design decisions, trade-offs

## Additional Directories

### adr/
Architecture Decision Records - documenting significant technical decisions.

### specs/
Implementation specifications and plans for upcoming work.

### bugs/
Known issues and investigation notes.

## Migration Status

| File | Target Location | Status |
|------|-----------------|--------|
| PROJECT_CONTEXT_AND_INTENT.md | explanation/ | pending |
| MODERN_DEVEX_SETUP.md | how-to/ | pending |
| paper_validation_report.md | reference/ | pending |
| ACTIVE_ISSUES.md | (root or bugs/) | pending |
