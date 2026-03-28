# CLAUDE.md — awesome-claude-code

This file provides guidance to AI assistants working in this repository.

## Project Overview

**awesome-claude-code** is a curated list of Claude Code resources (slash-commands, CLAUDE.md files, skills, workflows, tooling, hooks, etc.). It uses a CSV-driven generation pipeline to produce multiple README styles from a single data source.

- **Version**: 2.0.1
- **License**: CC-BY-NC-ND 4.0
- **Python**: 3.11+

## Repository Layout

```
awesome-claude-code/
├── THE_RESOURCES_TABLE.csv       # Master data source (single source of truth)
├── README.md                     # Generated root README (style: awesome)
├── README_ALTERNATIVES/          # 44+ generated README variants
├── acc-config.yaml               # Global README generation config
├── pyproject.toml                # Python project config + dependencies
├── Makefile                      # 29 targets for all common tasks
├── .pre-commit-config.yaml       # Pre-commit hooks (ruff, format, tests)
├── templates/
│   ├── categories.yaml           # Single source of truth for categories
│   ├── README_EXTRA.template.md
│   ├── README_CLASSIC.template.md
│   ├── README_AWESOME.template.md
│   └── footer.template.md
├── scripts/                      # 61 Python modules across 12 subsystems
├── tests/                        # 10 pytest test files
├── tools/                        # Utility scripts (README tree updater)
├── docs/                         # User and developer documentation
├── assets/                       # SVG badges, headers (~4.9 MB)
├── data/                         # Repo ticker CSV data
├── resources/                    # Downloaded resource files and CLAUDE.md examples
└── .github/
    ├── workflows/                # 13 GitHub Actions workflows
    └── ISSUE_TEMPLATE/           # Resource submission form
```

## Data Architecture

### THE_RESOURCES_TABLE.csv

The CSV is the **single source of truth** for all content. All READMEs are generated from it.

Columns: `ID, Display Name, Category, Sub-Category, Primary Link, Secondary Link, Author Name, Author Link, Active, Date Added, Last Modified, Last Checked, License, Description, Removed From Origin, Stale, Repo Created, Latest Release, Release Version, Release Source`

- **Resource IDs** follow the format `{category_prefix}-{8-char SHA256 hash}`
- Sorting order: Category → Sub-Category → Display Name
- The `Active` column controls whether a resource appears in READMEs

### templates/categories.yaml

Defines all 11 categories with IDs, prefixes, icons, subcategories, and ordering. This is used by every script that references categories. Do not hardcode category names in scripts — always load from this YAML.

## Development Workflows

### Environment Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -e ".[dev]"
pre-commit install
```

Locally, always use `venv/bin/python3`. In CI (`CI=true`), `python3` is used directly.

### Key Make Targets

| Command | Purpose |
|---|---|
| `make ci` | Full CI: format-check + mypy + test + docs-tree-check |
| `make test` | Run pytest tests |
| `make format` | Auto-fix linting/formatting with ruff |
| `make format-check` | Check formatting only (no fixes) |
| `make mypy` | Run mypy type checks |
| `make generate` | Sort CSV + generate all README variants |
| `make sort` | Sort THE_RESOURCES_TABLE.csv |
| `make validate` | Validate all resource URLs |
| `make validate-single URL=<url>` | Validate one URL |
| `make test-regenerate` | Delete READMEs, regenerate, verify no diff |
| `make coverage` | Run tests with coverage report |
| `make clean` | Remove caches and test artifacts |
| `make docs-tree` | Update file tree in docs/README-GENERATION.md |
| `make docs-tree-check` | Fail if docs tree is out of sync |

Always run `make ci` before committing. The `make generate` target runs `make sort` first automatically.

### README Generation Pipeline

```
THE_RESOURCES_TABLE.csv + templates/ + acc-config.yaml
        ↓
scripts/readme/generate_readme.py
        ↓
README.md (root style) + README_ALTERNATIVES/*.md (44 variants)
```

The four styles and their generators:
- **Extra** (`generators/visual.py`) — SVG-heavy visual style
- **Classic** (`generators/minimal.py`) — Plain markdown
- **Awesome** (`generators/awesome.py`) — Standard awesome-list style
- **Flat** (`generators/flat.py`) — 44 parameterized table views (11 categories × 4 sort modes)

Root style is controlled by `acc-config.yaml` → `readme.root_style`.

## Scripts Architecture

All scripts are importable as modules (e.g., `python -m scripts.readme.generate_readme`). The `scripts/` package follows a subsystem layout:

| Subsystem | Path | Purpose |
|---|---|---|
| README generation | `scripts/readme/` | Entry point + generators + helpers + markup |
| Validation | `scripts/validation/` | URL validation, single resource checks |
| Resources | `scripts/resources/` | CSV utilities, issue parsing, PR creation, download |
| Categories | `scripts/categories/` | Category management, add-category tool |
| IDs | `scripts/ids/` | Resource ID generation |
| Utilities | `scripts/utils/` | `repo_root.py`, `git_utils.py`, `github_utils.py` |
| Badges | `scripts/badges/` | Merge notifications, badge logic |
| Maintenance | `scripts/maintenance/` | Repo health checks, release data updater |
| Ticker | `scripts/ticker/` | Animated ticker SVG + data fetching |
| Graphics | `scripts/graphics/` | Logo/branding SVG generation |
| Testing | `scripts/testing/` | Regeneration cycle tests, TOC anchor validator |
| Archive | `scripts/archive/` | Deprecated scripts (excluded from linting/coverage) |

### Path Resolution

All scripts resolve the repo root by walking up to find `pyproject.toml` via `scripts.utils.repo_root.find_repo_root()`. Never assume a fixed working directory or use relative paths.

```python
from scripts.utils.repo_root import find_repo_root
REPO_ROOT = find_repo_root()
```

## Code Quality Standards

### Linting & Formatting (Ruff)

- **Line length**: 100 characters
- **Target**: Python 3.11
- **Rules**: E, W, F, I (isort), N (pep8-naming), UP (pyupgrade), B (bugbear), C4, SIM
- **Exceptions**: `scripts/*` exempt from E501; `__init__.py` exempt from F401; `scripts/archive/` excluded entirely
- Run `make format` to auto-fix; `make format-check` to verify only

### Type Checking (MyPy)

- Full type hints throughout (`scripts/` is PEP 561 compliant with `py.typed` marker)
- Run `make mypy` — covers `scripts/` and `tests/`
- `resources/` is excluded from mypy

### Testing (Pytest)

- Tests live in `tests/` directory
- Run with `make test` or `make coverage`
- `scripts/archive/` and `scripts/testing/test_regenerate_cycle.py` excluded from coverage
- Tests use a test CSV fixture, not `THE_RESOURCES_TABLE.csv` directly

## GitHub Workflows

13 automated workflows handle the full lifecycle:

| Workflow | Trigger | Purpose |
|---|---|---|
| `ci.yml` | Push / PR / dispatch | Format, mypy, tests |
| `validate-new-issue.yml` | Issue open/edit | Validate resource submissions |
| `submission-enforcement.yml` | Issues | Enforce submission format |
| `handle-resource-submission-commands.yml` | Issue comments | Process `/approve`, `/reject`, `/request-changes` |
| `validate-links.yml` | Scheduled / dispatch | Check all resource URLs |
| `update-github-release-data.yml` | Daily 3 AM UTC | Update release info in CSV |
| `check-repo-health.yml` | Weekly Mon 9 AM | Monitor repo health |
| `notify-on-merge.yml` | PR merge | Notify resource authors |
| `close-resource-pr.yml` | PR open | Auto-close direct resource PRs |

### Resource Submission Flow

1. User submits via GitHub Issues form (`recommend-resource.yml`)
2. Bot validates and labels `resource-submission`
3. Maintainer reviews, posts `/approve` comment
4. Bot creates PR, adds resource to CSV, regenerates README
5. On merge: author is notified via badge notification system

## Key Conventions

### Do Not Modify Generated Files Directly

`README.md` and all files in `README_ALTERNATIVES/` are generated. Edit the CSV or templates, then run `make generate`.

### Adding a New Resource

1. Add a row to `THE_RESOURCES_TABLE.csv` with a generated ID (`make generate-resource-id`)
2. Run `make sort` to sort the CSV
3. Run `make generate` to regenerate all READMEs
4. Run `make ci` to verify everything passes

### Adding a New Category

Use the interactive tool: `make add-category` or `make add-category ARGS='--name "..." --prefix x --icon 🎯'`

This updates `templates/categories.yaml` and regenerates relevant TOC assets.

### Modifying README Templates

Templates are in `templates/README_*.template.md`. After editing, run `make generate` and `make test-regenerate` to confirm the output is stable (no diff after a fresh regeneration).

### Environment Variables

- `GITHUB_TOKEN` — Set to avoid GitHub API rate limiting during validation or release data fetches
- `PYTHONPATH` — Should be set to repo root for module imports
- `CI=true` — Switches Python interpreter from `venv/bin/python3` to `python3`

## gitignore Notes

The root `CLAUDE.md` is listed in `.gitignore` (individual developer files are not committed). The exception `!resources/**/CLAUDE.md` allows CLAUDE.md files inside the `resources/` directory (these are curated examples).

`.claude/commands/evaluate-repository.md` is tracked; other `.claude/` contents are ignored.

## Dependencies

- **PyGithub** ≥ 2.1.1 — GitHub API access
- **PyYAML** ≥ 6.0.0 — Config and category loading
- **requests** ≥ 2.31.0 — HTTP link validation (dev)
- **python-dotenv** ≥ 1.0.0 — Environment config (dev)
- **ruff**, **mypy**, **pytest**, **pre-commit** — Dev tooling
