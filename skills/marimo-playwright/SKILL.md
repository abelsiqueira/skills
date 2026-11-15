---
name: marimo-playwright
description: Writes and debugs end-to-end browser tests for a marimo notebook served as an app, using pytest-playwright. Use when testing a marimo app through a real browser, or when such a test is flaky.
license: Apache-2.0
compatibility: Needs uv, pytest, pytest-playwright and a Chromium install.
---

# Marimo end-to-end tests with Playwright

`marimo run` serves a notebook as an app on a free port. A pytest fixture starts it and the `page` fixture from `pytest-playwright` drives it. This is a separate server, so a running `marimo edit` session on the same notebook is unaffected.

## Setup

```bash
uv add --dev playwright pytest-playwright
uv run playwright install chromium
```

Mark the browser tests, so a machine without Chromium still passes `pytest`. In `pyproject.toml`:

```toml
[tool.pytest.ini_options]
markers = ["e2e: browser tests (Playwright); run with `pytest -m e2e`"]
addopts = "-m 'not e2e' --tracing retain-on-failure --screenshot only-on-failure"
```

Put `pytestmark = pytest.mark.e2e` in each browser test module. Run them with `uv run pytest -m e2e`.

## Server fixture

Use a context manager, so a test can restart the server on the same data (call `serve()` twice on one `tmp_path`). A fixture wraps it for the common case. `--data-root` below stands for whatever arguments your notebook reads.

```python
import socket, subprocess, sys, time, urllib.request
from contextlib import contextmanager
from pathlib import Path

import pytest

PROJECT = Path(__file__).resolve().parents[2]  # the project that holds the notebook


def get_free_port():
    with socket.socket() as s:
        s.bind(("127.0.0.1", 0))
        return s.getsockname()[1]


def wait_until_ready(port, timeout=60):
    end = time.monotonic() + timeout
    while time.monotonic() < end:
        try:
            urllib.request.urlopen(f"http://127.0.0.1:{port}/health", timeout=1)
            return
        except OSError:
            time.sleep(0.3)
    raise TimeoutError(f"marimo not up on port {port}")


@contextmanager
def serve(*app_args, notebook="notebooks/app.py", cwd=PROJECT):
    port = get_free_port()
    cmd = [sys.executable, "-m", "marimo", "run", str(PROJECT / notebook),
           "--port", str(port), "--headless", "--", *app_args]
    proc = subprocess.Popen(cmd, cwd=cwd)
    try:
        wait_until_ready(port)
        yield f"http://127.0.0.1:{port}"
    finally:
        proc.terminate()
        try:
            proc.wait(10)
        except subprocess.TimeoutExpired:
            proc.kill()
            proc.wait()


@pytest.fixture
def app_url(tmp_path):
    with serve("--data-root", str(tmp_path)) as url:
        yield url
```

`--headless` stops marimo opening a browser tab. Arguments after `--` reach the notebook as `mo.cli_args()`, so point the app at test data that way.

## Finding widgets

marimo widgets are custom elements in shadow roots. Playwright locators pierce them. `document.querySelectorAll` in `page.evaluate` and `aria_snapshot()` do not. The `label=` of `mo.ui.text` is not an accessible name, so `get_by_role("textbox", name=...)` finds nothing. Match the rendered label text:

```python
def text_input(page, label):
    return page.locator(
        "marimo-text, marimo-text-area", has=page.get_by_text(label, exact=True)
    ).locator("input, textarea")
```

Buttons need `exact=True`, or `name="save"` also matches "save box" and raises a strict mode violation: `page.get_by_role("button", name=label, exact=True)`.

The searchable dropdown is the exception: its label names the button, so `page.get_by_label(label, exact=True)` works.

## Acting and waiting

- Call `box.blur()` after `box.fill(text)`, so marimo sends the value.
- Wait for the final state. Cells re-run one after another, so after a click a status text can change before the inputs reset. Typing too early lands in the old inputs and the kernel never sees it. Assert on the inputs themselves, e.g. `expect(text_input(page, "amount")).to_have_value("")` after a save.
- Assert that no cell raised: `expect(page.get_by_text("An internal error occurred")).to_have_count(0)`.
- Keep every user action in one `helpers.py` (`click_button`, `fill_text_input`, ...). Tests call these and use locators only to check what they see. A recorder or a retry can then hook in at one place.

Dragging on `mo.ui.matplotlib` needs scroll and coordinate care: see `reference/matplotlib-drag.md`.

## When a test fails or flakes

1. Read the trace and screenshot in `test-results/<test>/` first: `uv run playwright show-trace <trace.zip>`. `trace.zip` holds `trace.trace`, one JSON event per line (actions, logs, DOM snapshots).
2. Rerun the test once alone, and report both results.
3. A flake that passes alone often shows under load. Run it 20 times with every CPU busy (one `python3 -I -c 'while True: pass'` per core, killed afterwards). A race fix holds when 20 runs in a row pass.

## CI

Run the browser tests in their own job, so the unit job needs no browser:

```yaml
  e2e:
    runs-on: ubuntu-latest
    steps:  # checkout and setup-uv steps, then:
      - run: uv run --locked playwright install chromium
      - run: uv run --locked pytest -m e2e
```

In a last step with `if: failure()`, use `actions/upload-artifact` to upload `test-results/` as `e2e-test-results`, with `if-no-files-found: ignore`. Fetch it with `gh run download <run> -n e2e-test-results`.

Worked example: <https://github.com/nlesc-rags2riches/vox-paginae> (`tests/e2e/`).
