# Dragging on `mo.ui.matplotlib`

`mo.ui.matplotlib` renders a `<canvas>`. Drag a box on it with the mouse, using fractions of the canvas size.

```python
def drag_box(page, x0, y0, x1, y1):
    """Coordinates are fractions of the canvas."""
    canvas = page.locator("canvas").first
    canvas.scroll_into_view_if_needed()  # see the scroll gotcha below
    b = canvas.bounding_box()
    page.mouse.move(b["x"] + x0 * b["width"], b["y"] + y0 * b["height"])
    page.mouse.down()
    page.mouse.move(b["x"] + x1 * b["width"], b["y"] + y1 * b["height"], steps=10)
    page.mouse.up()
```

## Gotchas

- **Scroll before reading the position.**
  - Symptom: a rare failure, with the same wrong numbers each time. The box keeps its size and lands too high.
  - Cause (seen in marimo 0.25.1, not re-tested): on pointer down, `mo.ui.matplotlib` calls `focus()` without `preventScroll`. A canvas partly out of view makes the page jump, so the position read before the press is stale. A Playwright click that retries can also leave the page scrolled.
  - Fix: call `scroll_into_view_if_needed()` before `bounding_box()`. It helps only if the window is taller than the canvas. To check, set `document.getElementById('App').scrollTop` before the drag: `#App` scrolls, not the window.
- **Wait for the hint text before the first drag.** A box drawn before the page has been redrawn is dropped. Wait with `expect(page.get_by_text(hint)).to_be_visible()`, where `hint` is a text the app shows once it is ready.
- **The server needs no backend setup.** Inside a running notebook, marimo switches matplotlib to its own backend (`module://marimo._output.mpl`, built on Agg), whatever `MPLBACKEND` or `tests/conftest.py` says (checked with marimo 0.25.2). Put `matplotlib.use("Agg")` in `tests/conftest.py` only for unit tests that draw figures in the pytest process: some backends (macOS) double `fig.dpi` on a Retina screen, so canvas sizes differ from CI.
- **Fill the canvas to get exact pixels.** With `subplots_adjust(0, 0, 1, 1)` and `extent=(0, w, h, 0)`, the selection maps to image pixels: 0.2 of the canvas width comes back as 0.2 times the image width.
- **Give test images structure.** A constant image renders black, because `imshow` autoscales with `vmin == vmax`. Draw dark lines on a light page.
