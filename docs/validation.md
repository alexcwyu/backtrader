# Validation evidence: backtrader

Run date: 2026-10-12. Branch: `feature/ayu_develop`. Interpreter: `3.15.0 (main, Oct  9 2026, 15:11:47) [Clang 22.1.3 ]`.

| Check | Result |
|---|---|
| lock | passed |
| sync | passed |
| compile | passed |
| build | passed |
| tests | passed |

## Current result and compatibility decisions

- Python 3.15 installs, compiles, builds and passes the complete supplied function-based pytest suite.
- UTC constructors now use timezone-aware UTC and remove tzinfo to preserve the established naive-UTC public return convention.
- pytest python_classes=[] avoids collecting constructor-bearing simulation helper classes; all 82 supplied test functions are still collected.

Current unresolved checks:

All supplied checks recorded below passed.

## Latest full-suite output

```text
$ uv run --no-sync python -m pytest
........................................................................ [ 87%]
..........                                                               [100%]
82 passed in 104.41s (0:01:44)

```

## Exact commands and exit codes

Unsuccessful rows are retained as historical compatibility attempts; they are not claims that a current passing check remains blocked. Source-build probes use isolated temporary copies to preserve the selected repository environment.

| Command | Exit | Seconds | Evidence |
|---|---:|---:|---|
| `uv lock --upgrade --python 3.15` | 0 | 0.29 | [log](../../docs/maintenance-2026-10-12/backtrader-lock-3.15.log) |
| `uv sync --all-extras --all-groups --python 3.15 --no-build --no-install-project` | 0 | 0.08 | [log](../../docs/maintenance-2026-10-12/backtrader-wheel-probe-3.15.log) |
| `uv sync --all-extras --all-groups --python 3.15` | 0 | 0.76 | [log](../../docs/maintenance-2026-10-12/backtrader-sync-3.15.log) |
| `uv build --python 3.15` | 0 | 1.72 | [log](../../docs/maintenance-2026-10-12/backtrader-build.log) |
| `/Users/alex/.local/share/uv/python/cpython-3.15-macos-aarch64-none/bin/python3.15 -m compileall -q src` | 0 | 0.13 | [log](../../docs/maintenance-2026-10-12/backtrader-compile.log) |
| `uv run --no-sync python -m pytest tests` | 1 | 3.85 | [log](../../docs/maintenance-2026-10-12/backtrader-tests.log) |
| `uv run --no-sync python -m pytest` | 0 | 105.27 | [log](../../docs/maintenance-2026-10-12/backtrader-tests-utc-compatibility.log) |
| `uv run --no-sync python -m compileall -q src` | 0 | 0.13 | [log](../../docs/maintenance-2026-10-12/backtrader-compile-final.log) |
| `uv build --python 3.15` | 0 | 1.31 | [log](../../docs/maintenance-2026-10-12/backtrader-build-final.log) |
| `uv run --no-sync python -c import sys,importlib.metadata,json; print(sys.version); print(json.dumps({d.metadata["Name"]: d.version for d in importlib.metadata.distributions()},indent=2,sort_keys=True))` | 0 | 0.13 | [log](../../docs/maintenance-2026-10-12/backtrader-versions-final.log) |

The versions-final log records the exact interpreter patch release and every installed dependency version. The machine-readable [result](../../docs/maintenance-2026-10-12/backtrader-result.json) includes branch/default ancestry, status, artifacts and unresolved checks.

## Dependency floor refresh

| Before | After |
|---|---|

The table records the initial metadata refresh. Later compatibility constraints and vendor sources are explained above and the final authoritative requirements are in [pyproject.toml](../pyproject.toml) and [uv.lock](../uv.lock). No test cases were added, removed or disabled. Existing deprecated-API cases retain their assertions while explicitly checking the documented warning.
