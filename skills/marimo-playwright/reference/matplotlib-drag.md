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
- **Wait for each redraw before you drag.** A box drawn while the figure is being redrawn is dropped. This applies to the first drag after the page loads, and to the first drag after any action that redraws the figure, such as a save. Wait for something the redraw changes: a hint text with `expect(page.get_by_text(hint)).to_be_visible()`, or a new option in a dropdown.
- **The server needs no backend setup.** Inside a running notebook, marimo switches matplotlib to its own backend (`module://marimo._output.mpl`, built on Agg), whatever `MPLBACKEND` or `tests/conftest.py` says (checked with marimo 0.25.2). Put `matplotlib.use("Agg")` in `tests/conftest.py` only for unit tests that draw figures in the pytest process: some backends (macOS) double `fig.dpi` on a Retina screen, so canvas sizes differ from CI.
- **Make the image fill the canvas.** `mo.ui.matplotlib` maps the pointer across the whole canvas, so any margin around the image shifts every box. Remove the margins with `fig.subplots_adjust(0, 0, 1, 1)`. Give the figure the image's shape, `figsize=(height * w / h, height)`, so `imshow` adds no padding. If the shapes must differ, pass `aspect="auto"` to `imshow`, which stretches the image instead. With `extent=(0, w, h, 0)` as well, a selection comes back in image pixels: 0.2 of the canvas width is 0.2 times the image width.
- **Fix the colour range.** `imshow` scales colours to the image's own range. A flat image has no range, so it renders in the lowest colour, which is black with `cmap="gray"`. Pass `vmin` and `vmax`, for example 0 and 255 for 8-bit images, and a blank page shows as it is. Dark lines on a light test image still help a person read the trace screenshots.
- **Pixel size and on-screen size differ.** The canvas shows the figure's PNG at its pixel size, capped at the column width. Its `width` and `height` attributes give the PNG size, and `bounding_box()` gives the size on screen. `drag_box` uses `bounding_box()`, so fractions hold at any dpi. To keep the size on screen fixed when the dpi changes, wrap the widget in a fixed-width `div`: `mo.Html(f'<div style="width: 600px">{widget}</div>')`.

## Checking what was drawn

The canvas holds only pixels, so read them to check that a box is drawn. `locator.evaluate` runs on the element itself, so it reaches the canvas inside marimo's shadow root.

```python
def pixel(page, fx, fy):
    """[r, g, b] at fraction (fx, fy) of the canvas."""
    return page.locator("canvas").first.evaluate(
        """(c, [fx, fy]) => {
            const d = c.getContext("2d").getImageData(
                Math.round(fx * c.width), Math.round(fy * c.height), 1, 1).data;
            return [d[0], d[1], d[2]];
        }""",
        [fx, fy],
    )
```

Retry until a timeout, because the redraw is not instant.
