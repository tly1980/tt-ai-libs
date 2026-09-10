# Python — Project Setup

How to stand up and configure a Python project. For how to write the code once
it is set up, see [coding.md](coding.md).

**If you are an agent and were handed this file, read
[Bootstrap a New Project](#bootstrap-a-new-project) and run it top to bottom.**
Everything after that section is the reference material the steps point at.

Project-level `AGENTS.md` may override anything here — when it does, the
project wins.

## Bootstrap a New Project

The whole procedure is nine steps and ends with `make check` passing on an
empty skeleton. Do not skip the verify step — a skeleton that cannot lint is a
skeleton that will hide the first real failure.

### Step 0 — Inputs

You need three values before touching the filesystem. Take them from the
request if they are there; ask only for what is genuinely missing, and do not
block on the ones with an obvious default.

| Input | Default if unstated |
|---|---|
| Distribution name (`my-tool`, hyphens) | derived from the directory name |
| Import package (`my_tool`, underscores) | distribution name with `-` → `_` |
| Kind: library, CLI, or service | library, unless the request implies otherwise |

Python version is **not** an input — it is fixed at the current stable release
(see [Toolchain](#toolchain)). Do not ask, and do not take a caller's word for
a newer one.

### Step 1 — Scaffold

```bash
uv init --lib --name <dist-name> --python 3.12 .
```

`--lib` gives the `src/` layout, which every project here uses — including
CLIs and services. It also writes `.python-version` and a starter
`pyproject.toml` that step 2 replaces.

### Step 2 — `pyproject.toml`

Overwrite the generated file with this, substituting the two names. It is the
same in every project except the `[project]` block; copy the tool config
verbatim rather than reasoning about which rules to select. `uv init`
generates a `uv_build` backend — replacing it with `hatchling` is deliberate,
so the wheel builds the same way outside uv.

```toml
[project]
name = "<dist-name>"
version = "0.1.0"
description = "<one line>"
requires-python = ">=3.12"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/<package>"]

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

[tool.pytest.ini_options]
testpaths = ["tests"]
```

For a CLI, add a `[project.scripts]` entry — see
[Project Kinds](#project-kinds).

### Step 3 — Dev toolchain

```bash
uv add --dev ruff mypy pytest
```

Those three and nothing else by default. `pytest-cov` and `syrupy` are
opt-in — see [Toolchain](#toolchain).

### Step 4 — `Makefile`

Copy [the Makefile](#makefile) as-is. **Recipe lines must be tabs**, and this
is the step agents most often get wrong: `printf '%s\n'` does *not* expand
`\t`, and it will silently write a Makefile that make rejects. Use `%b`, or a
quoted heredoc containing real tab bytes.

Then verify before moving on — seven recipe lines must be tab-indented:

```bash
grep -cP '^\t' Makefile   # must print 7
```

### Step 5 — Layout

`uv init --lib` already created `src/<package>/__init__.py` and
`src/<package>/py.typed`. Add the test side, which it does not create:

```bash
mkdir -p tests/fixtures
touch tests/conftest.py
```

Check that `py.typed` is there and keep it — it ships the type information to
consumers, and without it a downstream `mypy` silently treats your package as
untyped. The rest of the tree is in [Layout](#layout).

### Step 6 — Smoke test

One real test, so `make test` proves the package is importable from the
installed wheel rather than from the working directory. Write it in the style
of [coding.md § Testing](coding.md#testing) — annotated, plain `assert`:

```python
# tests/test_import.py
import <package>


def test_package_imports() -> None:
    assert <package>.__name__ == "<package>"
```

Delete it once the project has tests that mean something.

### Step 7 — Config, secrets, ignores

Write `.env.example` — every key the project reads, with dummy values.

`uv init` writes a starter `.gitignore` covering `__pycache__/`, `*.py[oc]`,
build output and `.venv`. It misses four entries that matter here, so append
them:

```
.env
.pytest_cache/
.mypy_cache/
.ruff_cache/
```

The full list is in [Config and Secrets](#config-and-secrets).

### Step 8 — `AGENTS.md`

So the next agent to open the repo finds the rules without being handed them:

```markdown
# Agent Instructions

- Writing or changing Python code? Follow the rules in `<path>/coding.md`.
- Changing the toolchain, layout, or tool config? Follow `<path>/setup.md`.

## This project

<anything that overrides those rules, and why>
```

Leave the "This project" section empty rather than inventing overrides.

### Step 9 — Verify

```bash
make venv
make check
```

Both must pass before you report the project as set up. `make check` on a
fresh skeleton is fast and catches the things that actually go wrong: spaces
instead of tabs in the `Makefile`, a `packages = [...]` path that does not
match the real directory (hyphen vs underscore), an unannotated smoke test
tripping `strict` mypy, and a version skew against `.python-version`.

If it fails, fix the skeleton. Do not add an `ignore` entry or a `# noqa` to
get to green — see [coding.md § Lint Discipline](coding.md#lint-discipline).

### Done when

- [ ] `make check` passes from a clean clone
- [ ] `uv.lock` exists and is staged
- [ ] `.python-version`, `requires-python`, `target-version` and
      `[tool.mypy] python_version` all name the same version
- [ ] no `setup.py`, `setup.cfg`, `requirements.txt`, `.flake8`
- [ ] `.env` is ignored; `.env.example` is committed
- [ ] `AGENTS.md` points at these rules

Report which of these you verified rather than asserting the project is
"ready".

## Project Kinds

The layout, Makefile and tool config above are identical for all three. The
differences are small and local:

- **Library** — the default. Nothing to add.
- **CLI** — add an entry point and keep the parsing thin:
  ```toml
  [project.scripts]
  <dist-name> = "<package>.cli:main"
  ```
  `main()` parses arguments and calls into the package; the logic it calls
  stays importable and testable without a subprocess. A CLI is the one place
  `print()` is allowed, and only for its own output — see
  [coding.md § Errors and Logging](coding.md#errors-and-logging).
- **Service** — add the framework as a real dependency (`uv add fastapi`),
  keep the app factory in `src/<package>/app.py`, and add a `run` target below
  the standard five in the `Makefile`.

Adding a runtime dependency is `uv add <pkg>`, never a hand-edit of
`[project.dependencies]` — the lock file has to move with it.

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
  .env.example
  AGENTS.md
  src/<package>/        ## importable code, src layout
    __init__.py
    py.typed            ## marks the package as typed for consumers
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

`ruff` is both the formatter and the linter. Its config, along with mypy's,
lives in `pyproject.toml` — the block in step 2 of
[Bootstrap a New Project](#bootstrap-a-new-project) is the same in every
project. Notes on it:

- Keep `target-version` and `[tool.mypy] python_version` in sync with
  `.python-version` and `requires-python`.
- `strict = true` is the mypy baseline, not an aspiration. If a third-party
  package has no stubs, silence that import specifically:
  ```toml
  [[tool.mypy.overrides]]
  module = ["untyped_pkg.*"]
  ignore_missing_imports = true
  ```
  That is a narrow, named exception — never `ignore_errors` over your own code.
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

## CI

If the project has a GitHub remote, add `.github/workflows/check.yml`. It runs
the same `make check` a developer runs — CI should never have its own idea of
what passing means:

```yaml
name: check
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true
      - run: make venv
      - run: make check
```

`uv` reads `.python-version`, so the workflow does not name a version — one
less place to fall out of sync.

## What to Avoid

- `pip`, `poetry`, `conda`, or `venv` directly — `uv` owns the environment.
- Chasing the newest Python release. Wait until it is the established stable
  version and the ecosystem has caught up, then bump this file.
- Installing black, isort, or flake8 alongside ruff — they will fight it.
- Committing `.venv/`, `__pycache__/`, `.pytest_cache/`, or `*.pyc`.
- Scaffolding a `tests/` tree with no test in it. One passing smoke test at
  setup time is what proves the layout and the packaging config agree.
- Reporting setup as done without running `make check`.
