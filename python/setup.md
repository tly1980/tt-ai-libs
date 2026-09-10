# Python — Project Setup

How to stand up and configure a Python project. For how to write the code once
it is set up, see [coding.md](coding.md).

Project-level `AGENTS.md` may override anything here — when it does, the
project wins.

## Toolchain

| Task | Command | Never |
|---|---|---|
| Install / sync deps | `uv sync` | `pip install`, `poetry install` |
| Add a dependency | `uv add <pkg>` (`--dev` for tooling) | hand-editing `[project.dependencies]` |
| Run anything | `uv run <cmd>` | activating `.venv` by hand |
| Format | `uv run ruff format .` | `black`, `isort` |
| Lint | `uv run ruff check --fix .` | `flake8`, `pylint` |
| Type check | `uv run mypy .` | skipping it |
| Test | `uv run pytest` | bare `pytest` |

- Track a **stable, widely-supported** Python release — never the newest one, a
  release candidate, or a pre-release. Check the status of each version at
  [devguide.python.org/versions](https://devguide.python.org/versions/); pick
  one in `security` or `bugfix` status, not `prerelease` or `feature`.
  **Today that is 3.12**: set `requires-python = ">=3.12"` in `pyproject.toml`
  and pin the same version in `.python-version` so `uv` provisions it
  consistently.
- Every project installs the same dev toolchain — nothing else is needed to run
  `make lint` and `make test`:
  ```
  uv add --dev ruff mypy pytest
  ```
  `ruff` covers formatting *and* linting (it replaces black, isort, flake8 and
  its plugins — do not install those separately). Add `pytest-cov` only if the
  project reports coverage, and `syrupy` only if it uses snapshot testing.
- Commit `uv.lock`. It is the source of truth for what actually got installed.
- `pyproject.toml` is the only config file — no `setup.py`, `setup.cfg`,
  `requirements.txt`, or `.flake8`.

## Layout

```
<repo>/
  Makefile
  pyproject.toml
  uv.lock
  .python-version
  src/<package>/        ## importable code, src layout
    __init__.py
  tests/                ## mirrors src/<package>/ structure
    conftest.py         ## pytest fixtures (code)
    fixtures/           ## static test data (files)
    __snapshots__/      ## committed snapshots, written by syrupy
```

- Use the `src/` layout so tests import the installed package, not the working
  directory — catches missing-file-in-wheel bugs before they ship.
- One `tests/test_<module>.py` per `src/<package>/<module>.py`.
- The `tests/conftest.py` vs `tests/fixtures/` split matters — see
  [coding.md § Fixtures](coding.md#fixtures).

## Makefile

Every project gets a `Makefile` with the same target names, so the entry point
is identical whether you are a human, an agent, or CI. Copy this as-is:

```make
.PHONY: venv lint format test check

venv:          ## create .venv and install all deps
	uv sync --all-extras --dev

lint:          ## check only — never writes
	uv run ruff check .
	uv run ruff format --check .
	uv run mypy .

format:        ## auto-fix what can be auto-fixed
	uv run ruff check --fix .
	uv run ruff format .

test:
	uv run pytest

check: lint test   ## what CI runs
```

- `lint` must never modify files — CI runs it and so do pre-commit checks.
  `format` is the only target allowed to write.
- Add project-specific targets (`run`, `migrate`, `docker`) below these, but
  never rename the five above.
- Recipe lines are indented with **tabs**, not spaces — make will fail otherwise.

## Tool Configuration

`ruff` is both the formatter and the linter. Configure it in `pyproject.toml` —
same block in every project:

```toml
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = [
  "F",    # pyflakes — real bugs
  "E",    # pycodestyle errors
  "I",    # import sorting
  "N",    # pep8 naming
  "UP",   # pyupgrade — modern syntax for target-version
  "B",    # bugbear — mutable defaults, shadowed loop vars
  "C4",   # comprehensions
  "SIM",  # simplify
  "PTH",  # force pathlib over os.path
  "G",    # logging format — no f-strings in log calls
  "RUF",
]
ignore = ["E501"]  # the formatter owns line length

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]  # asserts are the point in tests

[tool.mypy]
python_version = "3.12"
strict = true
```

- Keep `target-version` and `[tool.mypy] python_version` in sync with
  `.python-version` and `requires-python`.
- Adding a rule to `ignore` is a project-wide decision — raise it rather than
  doing it silently to make a build pass. How to handle individual findings is
  covered in [coding.md § Lint Discipline](coding.md#lint-discipline).

## Config and Secrets

- Read config from environment variables, loaded from a local `.env`.
- `.env` is **git-ignored**. Commit a `.env.example` listing every key with
  dummy values, so a fresh clone knows what it needs.
- `.gitignore` must cover at least: `.venv/`, `.env`, `__pycache__/`, `*.pyc`,
  `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`.
- Never commit a real credential, token, hostname, or IP — see
  [coding.md § Secrets](coding.md#secrets).

## What to Avoid

- `pip`, `poetry`, `conda`, or `venv` directly — `uv` owns the environment.
- Chasing the newest Python release. Wait until it is the established stable
  version and the ecosystem has caught up, then bump this file.
- Installing black, isort, or flake8 alongside ruff — they will fight it.
- Committing `.venv/`, `__pycache__/`, `.pytest_cache/`, or `*.pyc`.
