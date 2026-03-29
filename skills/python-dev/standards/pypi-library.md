# Standard: PyPI Library Publishing

> **Audience:** AI agents writing, reviewing, or packaging Python libraries for PyPI.
> **Prerequisite:** Read `skills/python-dev/standards/idiomatic-python.md` for general Python rules.

## Core Principle

A published Python library is a **public contract**. Use modern packaging (`pyproject.toml`, no `setup.py`), type annotations everywhere, and automated publishing via trusted publishers. Humans decide *what* to release — CI does the rest.

---

## 1. pyproject.toml — Single Source of Config

**Rule:** All project metadata, build config, and tool config live in `pyproject.toml`. No `setup.py`, `setup.cfg`, or `requirements.txt` for library dependencies.

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-library"
dynamic = ["version"]
description = "A short description"
readme = "README.md"
license = "MIT"
requires-python = ">=3.12"
authors = [{ name = "John Azariah", email = "john@example.com" }]
classifiers = [
    "Development Status :: 4 - Beta",
    "Programming Language :: Python :: 3.12",
    "Typing :: Typed",
]
dependencies = [
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-cov", "mypy", "ruff"]

[tool.hatch.version]
path = "src/my_library/__init__.py"
```

### Guidelines

- **`hatchling`** as build backend — modern, fast, good defaults. Alternatives: `flit_core`, `setuptools` (legacy).
- **`dynamic = ["version"]`** — version lives in `__init__.py`, hatch reads it.
- **`requires-python = ">=3.12"`** — match the project's minimum.
- **No `requirements.txt`** for library deps — use `[project.dependencies]`. Keep `requirements.txt` only for pinned app deployments.

---

## 2. Versioning via Git Tags

**Rule:** Use `hatch-vcs` (or `setuptools-scm`) to derive version from git tags. The tag is the single source of truth.

```toml
[build-system]
requires = ["hatchling", "hatch-vcs"]
build-backend = "hatchling.build"

[tool.hatch.version]
source = "vcs"

[tool.hatch.build.hooks.vcs]
version-file = "src/my_library/_version.py"
```

### Flow

1. Tag `v1.0.0` → builds produce `1.0.0`
2. Commit after tag → builds produce `1.0.1.dev1` (auto dev version)
3. Tag `v1.1.0` → builds produce `1.1.0`

### Guidelines

- **Never hardcode version** in pyproject.toml when using `hatch-vcs`.
- **Tag prefix `v`** — matches the release process convention.
- **`_version.py`** is auto-generated, add it to `.gitignore`.

---

## 3. Type Annotations and py.typed

**Rule:** Every public function and class must have full type annotations. Include a `py.typed` marker file for PEP 561 compliance.

```
src/my_library/
├── __init__.py
├── _version.py
├── py.typed          ← empty file, signals type info available
├── core.py
└── types.py
```

### Guidelines

- **`py.typed`** — required for mypy/pyright to recognise the package as typed.
- **`__all__`** in `__init__.py` — explicitly control the public API surface.
- **No `Any`** in public signatures — be specific.
- **`Protocol`** for capability interfaces (Tagless-Final pattern).
- **`@overload`** for functions with different return types based on input.

---

## 4. Exports and API Surface

**Rule:** Control what's public via `__all__` in every module. Re-export from `__init__.py` for a clean import API.

```python
# src/my_library/__init__.py
"""My Library — a short description."""

from my_library.core import process, transform
from my_library.types import Config, Result

__all__ = ["Config", "Result", "process", "transform"]
```

### Guidelines

- **Underscore-prefixed** modules (`_internal.py`) for implementation details.
- **Flat imports** — consumers should write `from my_library import Config`, not `from my_library.types import Config`.
- **Re-export only stable API** — internal helpers stay in underscore modules.

---

## 5. Build and Test

```bash
# Build
python -m build

# Test
pytest --cov=my_library --cov-report=term-missing

# Type check
mypy src/

# Lint
ruff check src/ tests/
```

---

## 6. CI Workflow — publish-pypi.yml

Uses PyPI trusted publishers (no API tokens needed):

```yaml
name: Publish to PyPI
on:
  push:
    tags: ['v[0-9]+.[0-9]+.[0-9]+']

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      id-token: write        # Required for trusted publishing
      contents: read
    environment:
      name: pypi
      url: https://pypi.org/p/my-library
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0     # hatch-vcs needs full history
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install build
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
```

### Guidelines

- **Trusted publishers** over API tokens — more secure, no secrets to manage.
- **Configure on PyPI** — link your GitHub repo as a trusted publisher in the PyPI project settings.
- **`fetch-depth: 0`** — hatch-vcs needs the full git history for version derivation.

---

## Self-Check Table

| # | Smell | Fix |
|---|-------|-----|
| 1 | `setup.py` or `setup.cfg` present | Migrate to `pyproject.toml` |
| 2 | Hardcoded version string | Use `hatch-vcs` — version from git tags |
| 3 | Missing `py.typed` marker | Add empty `py.typed` to package root |
| 4 | Missing `__all__` in public modules | Define explicit exports |
| 5 | `Any` in public function signatures | Use specific types or `Protocol` |
| 6 | `requirements.txt` for library dependencies | Use `[project.dependencies]` in pyproject.toml |
| 7 | API token in CI secrets for PyPI | Switch to trusted publishers |
