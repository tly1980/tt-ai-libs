# Python — Coding Rules

How to write Python in this codebase. For toolchain, project layout, Makefile
and tool configuration, see [setup.md](setup.md).

Project-level `AGENTS.md` may override anything here — when it does, the
project wins.

## First Principle

**Readability and maintainability beat cleverness and brevity, every time.**

Code is read far more often than it is written, and the next reader is usually
someone with no context — a teammate months from now, or an agent seeing the
file for the first time. Optimise for that reader.

- Fewer lines is not a goal. Clear lines are.
- If a reviewer has to pause and decode a line, rewrite it — even if it was
  correct and even if it was shorter.
- Prefer the obvious construct over the idiomatic-but-dense one. Boring code
  is a feature.
- When a rule below conflicts with readability in a specific case, readability
  wins — say so in the PR rather than following the rule off a cliff.

## Typing

- Annotate every function signature — parameters and return type.
- Use builtin generics (`list[str]`, `dict[str, int]`, `X | None`) — not
  `typing.List`, `typing.Optional`.
- `Any` is an escape hatch, not a default. If you reach for it, leave a comment
  saying why.
- Prefer `@dataclass` (or Pydantic where already in use) over passing bare
  dicts between functions.

## Style

- Ruff defaults, line length 88. Let the formatter decide — never hand-wrap.
- Names: `snake_case` functions/variables, `PascalCase` classes,
  `UPPER_SNAKE` module-level constants, `_leading_underscore` for private.
- Prefer pure functions. Reach for a class only when there is real state to hold.
- Use `pathlib.Path`, not `os.path`.
- f-strings for interpolation — except in logging calls (see below).

## No One-Liners

Compressing several steps onto one line saves a line and costs every future
reader. Give each step a name and its own line:

```python
# no — one dense line, unnamed intermediates, unreadable diff when it changes
users = [u for u in (json.loads(r.text) for r in resps) if u.get("active") and u["age"] > 18]

# yes — each step named, each independently debuggable
payloads = [json.loads(r.text) for r in resps]
adults = [u for u in payloads if u["age"] > 18]
users = [u for u in adults if u.get("active")]
```

Specifically avoid:

- Nested or multi-clause comprehensions. One `for` and at most one `if` per
  comprehension — beyond that, write the loop.
- Statements joined with `;`.
- Single-line bodies: `if x: return y`, `for i in xs: f(i)`. Always break.
- `lambda` assigned to a name — that is just a `def` with worse tracebacks.
- Deeply chained calls on one line — split with intermediate variables whose
  names explain what each stage produced.
- Walrus (`:=`) used to save a line rather than to avoid a real recomputation.
- Nested ternaries. One `x if cond else y` is fine; two is an `if` block.

An intermediate variable is not waste — it is a name, a breakpoint, and a
one-line diff when the logic changes.

## Lint Discipline

The ruff and mypy config lives in `pyproject.toml` — see
[setup.md § Tool Configuration](setup.md#tool-configuration).

- Fix lint findings; do not silence them.
- A `# noqa` needs a specific rule code and a trailing comment explaining why —
  bare `# noqa` is not allowed.
- Same for `# type: ignore[code]`. Never blanket `# type: ignore`.
- `make lint` passing is the bar for a PR, not a suggestion.

## Errors and Logging

- Raise specific exceptions (`ValueError`, `FileNotFoundError`, or a project
  exception type). Never raise bare `Exception`.
- Never write a bare `except:` or a silent `except Exception: pass`. Catch what
  you can actually handle.
- Validate at boundaries — CLI args, HTTP payloads, file contents. Trust
  internal callers.
- `logging` module, one `logger = logging.getLogger(__name__)` per module. Never
  `print()` outside a CLI's own output.
- Use lazy log formatting: `logger.info("loaded %s rows", n)` — not an f-string.

## Secrets

- Never hard-code credentials, tokens, hostnames, or IPs — read them from the
  environment. Setup side of this is in
  [setup.md § Config and Secrets](setup.md#config-and-secrets).
- Never log a secret, a full auth header, or a whole request body that might
  carry one.
- Scrub real credentials and customer data out of anything committed to
  `tests/fixtures/`.

## Testing

- `pytest` only — no `unittest.TestCase` classes.
- Plain `assert`; use `pytest.raises` for expected failures.
- `@pytest.mark.parametrize` instead of loops over cases inside one test.
- Mock at the network/subprocess boundary only. Do not mock your own modules
  just to make a test pass.

### Fixtures

Two different things share the name — keep them apart:

- **`tests/conftest.py`** — pytest fixtures, i.e. *code*: setup/teardown,
  fake clients, factory helpers. Scope them (`function` by default) and put a
  fixture in the narrowest `conftest.py` that needs it, not the root one.
- **`tests/fixtures/`** — static test *data*: sample JSON payloads, CSVs,
  recorded API responses, config files. Never generate these at test time.

Rules for `tests/fixtures/`:

- Reference them by path relative to the test file, never by CWD:
  ```python
  FIXTURES = Path(__file__).parent / "fixtures"
  payload = json.loads((FIXTURES / "order_created.json").read_text())
  ```
- Name a file after the case it represents (`order_created.json`,
  `invoice_missing_tax.csv`) — not `test1.json`, `data.json`.
- Keep them small. A fixture is an illustrative example, not a prod dump — trim
  it to the fields the test actually exercises and scrub any real credentials,
  customer names, or tokens before committing.
- Use `tmp_path` for anything the test *writes*; `tests/fixtures/` is read-only.

### Snapshot Testing

Use `syrupy` (`uv add --dev syrupy`) when the assertion is "this large
structured output did not change" — rendered templates, CLI output, serialized
API responses, generated SQL.

```python
def test_renders_invoice(snapshot):
    assert render_invoice(order) == snapshot
```

- Snapshots are **committed and code-reviewed**. A snapshot diff in a PR is the
  actual behavioural change under review — read it like a code diff.
- Reach for a snapshot only when hand-writing the expected value would be
  unreadable. For a 3-field dict, assert the fields — a snapshot there hides
  what the test is actually checking.
- Never assert a snapshot over unstable values (timestamps, UUIDs, dict
  ordering, absolute paths). Freeze or scrub them first, or the test flakes.
- `--snapshot-update` is a deliberate act: run it, then read every changed
  snapshot before staging. Blanket-regenerating to get green is how a broken
  behaviour becomes the new expected value.
- Prefer one snapshot per behaviour over one giant snapshot of everything — a
  huge snapshot turns any change into an unreviewable diff.

## What to Avoid

- One-liners that trade clarity for line count — see [No One-Liners](#no-one-liners).
- Clever code. If it needs a comment to explain *how* it works, rewrite it
  simpler instead of writing the comment.
- Mutable default arguments (`def f(x=[])`).
- Wildcard imports (`from x import *`).
- `sys.path` manipulation to make an import work — fix the package layout.
- Comments that restate the code. Comment the *why*, never the *what*.
