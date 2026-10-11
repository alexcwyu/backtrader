# backtrader: portable verification records

Full raw logs are retained in the workspace maintenance evidence, not distributed here. Hashes identify those original logs. A successful build or selected test subset does not imply a full test-suite pass. Commands containing <workspace> describe the original execution environment.

[Machine-readable records](verification.json) · [Repository validation summary](../validation.md)

## Executed commands

| Check | Exit | Duration | Command | Outcomes |
| --- | ---: | ---: | --- | --- |
| build | 0 | 1.72s | `uv build --python 3.15` | Successfully built dist/backtrader-1.9.78.123.tar.gz<br>Successfully built dist/backtrader-1.9.78.123-py3-none-any.whl |
| build-final | 0 | 1.31s | `uv build --python 3.15` | Successfully built dist/backtrader-1.9.78.123.tar.gz<br>Successfully built dist/backtrader-1.9.78.123-py3-none-any.whl |
| compile | 0 | 0.13s | `/Users/alex/.local/share/uv/python/cpython-3.15-macos-aarch64-none/bin/python3.15 -m compileall -q src` | See exit status; no test-count assertion. |
| compile-final | 0 | 0.13s | `uv run --no-sync python -m compileall -q src` | See exit status; no test-count assertion. |
| lock-3.15 | 0 | 0.29s | `uv lock --upgrade --python 3.15` | See exit status; no test-count assertion. |
| sync-3.15 | 0 | 0.76s | `uv sync --all-extras --all-groups --python 3.15` | See exit status; no test-count assertion. |
| tests | 1 | 3.85s | `uv run --no-sync python -m pytest tests` | 76 failed, 6 passed in 3.24s |
| tests-utc-compatibility | 0 | 105.27s | `uv run --no-sync python -m pytest` | 82 passed in 104.41s (0:01:44) |
| versions-final | 0 | 0.13s | `uv run --no-sync python -c import sys,importlib.metadata,json; print(sys.version); print(json.dumps({d.metadata["Name"]: d.version for d in importlib.metadata.distributions()},indent=2,sort_keys=True))` | See exit status; no test-count assertion. |
| wheel-probe-3.15 | 0 | 0.08s | `uv sync --all-extras --all-groups --python 3.15 --no-build --no-install-project` | See exit status; no test-count assertion. |
