# Pixel Art Editor — Implementation Plan

> Research completed 2026-05-10. Concrete recommendations for a vanilla JS Canvas pixel art editor.

---

## Architecture Overview

- **Source canvas** (offscreen): holds the actual pixel grid at native resolution (e.g. 16x16 to 256x256). All drawing operations write here.
- **Display canvas** (visible in DOM): scales and renders the source canvas with `imageSmoothingEnabled = false` for pixel-perfect display. Handles viewport transform (zoom + pan).
- **UI layer**: toolbar, palette panel, frame strip — plain HTML/CSS, not canvas. Keeps code simpler and more accessible.
- **No frameworks**. Vanilla JS only. No npm / build step. Single HTML file with `<script>` tags and `<style>` block preferred.

---

## 1. Color Palette Management

### Rendering Strategy
- **Offscreen canvas + fillRect**: Build the palette once on an offscreen canvas using `fillRect` per swatch. Then `drawImage` to the visible canvas. Rebuild only when the active palette changes or a custom color is added/removed.
- Swatch size: 20x20px with 2px gap. Grid layout calculated from available width.
- Selection highlight: redraw the offscreen canvas with a thick border (2px white stroke) on the selected swatch.

### Preset Palettes

**Game Boy (4 colors):**
```
#0F380F  #306230  #8BAC0F  #9BBC0F
```

**PICO-8 (16 colors):**
```
#000000  #1D2B53  #7E2553  #008751
#AB5236  #5F574F  #C2C3C7  #FFF1E8
#FF004D  #FFA300  #FFFF27  #00E436
#29ADFF  #83769C  #FF77A8  #FFCCAA
```

**NES (56 colors):**
```
#626262 #001FB0 #2400AA #4F008E #70006A #78003E #78000E #6A0000
#570000 #3F0000 #240000 #0D0000 #000000 #000000 #000000 #000000
#A8A8A8 #002FFF #4500FF #7700FF #9900D6 #A80079 #A80021 #9A0000
#7F0000 #5E0000 #3A0000 #1B0000 #000000 #000000 #000000 #000000
#FFFFFF #0062FF #7C00FF #B600FF #E000D2 #F00070 #EE0015 #D20000
#A80000 #7F0000 #540000 #2F0000 #050000 #000000 #000000 #000000
#FFFFFF #7FB8FF #AC74FF #E174FF #FF74D1 #FF7471 #FF740D #D27C00
#9B6E00 #695800 #3E4000 #1A2A00
```

### Color Picking
- Click on the palette canvas → map screen coordinates to swatch index using `canvas.getBoundingClientRect()` and grid math.
- Return early if click is in the gap between swatches.

### Custom Colors
- "+" button → `<input type="color">` → append to custom palette → rebuild offscreen canvas.
- Right-click swatch to remove (on custom palettes only).
- Persist custom palettes to `localStorage` as JSON.

### Implementation: `PaletteManager` class (see appendix)

---

## 2. Zoom and Pan

### Core State
```js
{ scale: number,  offsetX: number,  offsetY: number }
```
- `scale`: 1–32 (integer preferred for pixel-perfect rendering; clamp after operations).
- `offsetX`, `offsetY`: world coordinates at canvas origin (top-left).

### Coordinate Mapping
```
worldX = screenX / scale + offsetX
worldY = screenY / scale + offsetY
screenX = (worldX - offsetX) * scale
screenY = (worldY - offsetY) * scale
```

### Zoom by Mouse Wheel
- Zoom centered on cursor: compute cursor world position before zoom, apply new scale, recompute offset so cursor stays fixed.
- Formula: `offset = cursorWorld - cursorScreen / newScale`
- Accumulate fractional wheel deltas; apply when threshold reached, then snap to integer scale.

### Zoom Buttons (+ / -)
- Multiply/divide scale by 2 (or 1.5 for fine control).
- Keep canvas CENTER fixed: use viewport center world coords instead of cursor.

### Pinch-to-Zoom (Mobile)
- Track two `pointerId`s. On pointerdown with second finger, compute initial distance and midpoint.
- On pointermove: `newScale = oldScale * (currentDist / initialDist)`.
- Keep pinch midpoint fixed in world space (same formula as cursor zoom).
- Clamp to 1–32.

### Panning (Drag)
- mousedown → record start position + start offset.
- mousemove → `offset = startOffset + (startMouse - currentMouse) / scale`.
- Single-touch pan on mobile when only one finger active.

### Efficient Re-Rendering
- Source canvas holds pixel art at native resolution.
- Display canvas renders: `ctx.drawImage(sourceCanvas, offsetX, offsetY, displayW/scale, displayH/scale, 0, 0, displayW, displayH)`.
- Single `clearRect` + `drawImage` per frame. GPU-accelerated, handles any zoom level.
- High-DPI: multiply display canvas backing store by `devicePixelRatio`, scale context accordingly.

### Viewport Clamping
- Clamp `offsetX`/`offsetY` so the source canvas always fills at least part of the viewport. Don't allow panning into empty space beyond the pixel art.

---

## 3. Symmetry / Mirror Drawing

### Architecture
- All mirroring happens on the **source canvas** (pixel grid) during drawing.
- For each pixel drawn, compute and draw the mirrored pixel(s) immediately.
- This guarantees real-time mirroring with zero visible lag.

### Horizontal Mirror (Left-Right)
- Axis: `axisX = width / 2`
- **Even width** (axis between columns): every pixel gets a distinct mirror at `mirrorX = 2 * axisX - x`
- **Odd width** (axis on center column): if `x === Math.floor(axisX)`, draw only once (on-axis). Otherwise mirror.
- Clamp mirrored coordinates to `[0, width-1]`.

### Vertical Mirror
- Same logic on Y axis: `mirrorY = 2 * axisY - y`

### Quadrant Symmetry (Both)
- Up to 4 copies per pixel: original + horizontal + vertical + diagonal.
- Deduplicate: use a Set of `"x,y"` keys or just let overwrites happen (harmless for fillRect).

### Performance
- Use **ImageData** for batch pixel updates during drawing:
  1. Hold `ImageData` object for source canvas.
  2. On each pointer move event: set RGBA values for all pixels (original + mirrors) directly in `data` array.
  3. `ctx.putImageData()` → single bulk transfer.
  4. Copy to display canvas via `drawImage`.
- This scales to 256x256 canvases at 60fps.
- Alternative for small canvases (≤64x64): `fillRect(1,1)` per pixel — simpler, still fast enough.

### Line Drawing with Mirroring
- Use Bresenham's line algorithm between last and current pointer position.
- For each step in the line, draw original + mirrored pixels.
- Ensures continuous strokes even with fast pointer movement.

### UI
- Toggle buttons: "Mirror X", "Mirror Y", "Quadrant".
- Switching mode doesn't retroactively apply to existing pixels — only affects new strokes.

---

## 4. Animation Frames

### Frame Storage
- **Array of OffscreenCanvas**: `const frames = [new OffscreenCanvas(w, h), ...]`
- OffscreenCanvas enables GPU-accelerated `drawImage` for playback and thumbnails.
- NOT ImageData arrays — slower, no GPU compositing.

### Editing Workflow
- Single visible editing canvas. On frame switch:
  1. Save current editing canvas content to `frames[currentIndex]` via `drawImage`.
  2. Load new frame content onto editing canvas via `drawImage`.

### Frame List (Thumbnails)
- Generate 32x32 thumbnail canvases for each frame: `drawImage(frameCanvas, 0, 0, 32, 32)`.
- Cache thumbnails. Mark dirty on edit; regenerate only for changed frames.
- Display as a horizontal strip above or below the main canvas. Active frame gets a highlighted border.

### Frame Operations
- **Add**: `frames.push(new OffscreenCanvas(w, h))` → clear to transparent.
- **Delete**: `frames.splice(index, 1)` → null reference for GC.
- **Duplicate**: `drawImage(source, 0, 0)` onto new OffscreenCanvas → insert after source.
- **Reorder**: drag-and-drop or arrow buttons → `splice` to move.

### Onion Skinning
- Precompute ghost canvases for adjacent frames when current frame changes.
- Ghost creation: `ctx.globalCompositeOperation = 'source-atop'` + fill with semi-transparent color → tints only non-transparent pixels.
- Render on editing canvas: draw previous ghost at 50% alpha, current frame at 100%, next ghost at 50%.

### Animation Preview
- Playback loop: `requestAnimationFrame` with delta-time FPS control.
- Track accumulated time; advance frame when `accumulated >= 1000/fps`.
- Disable editing interactions during playback. Show "Stop" button.
- FPS slider: 1–60. Default 10 FPS (common for pixel art).
- Render: `ctx.clearRect` + `ctx.drawImage(frames[currentFrame], 0, 0)` each tick.

### Memory
- 128x128 canvas = ~65 KB per frame. 64 frames = ~4 MB. Well within browser budgets.
- Thumbnails: negligible (~4 KB each).
- Ghost canvases: recomputed on demand, GC'd when no longer referenced.

---

## 5. PNG Export / Import

### Export
```js
function exportPNG(sourceCanvas, filename) {
  sourceCanvas.toBlob(blob => {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
  }, 'image/png');
}
```
- Use `toBlob()` (async, memory-efficient) not `toDataURL()` (sync, base64 bloat).
- PNG encoder is lossless — no anti-aliasing, no compression artifacts. Exact pixel data preserved.
- Transparency: alpha channel preserved as-is. No special handling needed.

### Import
```
File input → FileReader → Image element → offscreen canvas → getImageData
```
1. `<input type="file" accept="image/png">` → File API.
2. `FileReader.readAsDataURL(file)` → set as `img.src`.
3. On `img.onload`: draw to offscreen canvas at naturalWidth/naturalHeight with `imageSmoothingEnabled = false`.
4. `ctx.getImageData(0, 0, w, h)` → raw pixel array.
5. Map RGBA to editor's internal format. Optionally match to nearest palette color.

### Handling Scaled-Up PNGs
- Common case: 16x16 pixel art scaled up 16x to 256x256.
- Provide "Downsample Factor" dropdown (Off, 2x, 4x, 8x, 16x). Default: Off (use raw dimensions).
- If factor is set: draw source at `origW/factor × origH/factor` with `imageSmoothingEnabled = false` → nearest-neighbor picks corner pixel of each block.
- Validate imported size against editor's max dimensions.

### Optional: PNG Metadata
- Embed editor state (palette, canvas size, scale factor) in PNG `tEXt` chunks for lossless round-tripping.
- Requires manual chunk encoding via `DataView` on blob's ArrayBuffer. Nice-to-have, not v1.

---

## 6. Mobile Touch Support

### Pointer Events API (Unified Mouse + Touch)
- Use `pointerdown`, `pointermove`, `pointerup`, `pointercancel` for all input.
- Use `setPointerCapture(e.pointerId)` on pointerdown — ensures canvas receives all subsequent moves even if finger leaves element.
- Use `e.getCoalescedEvents()` to get high-frequency events on pointermove for smooth strokes.

### Preventing Default Browser Behaviors
```css
#pixel-canvas {
  touch-action: none;   /* disable all browser gestures */
  user-select: none;    /* prevent text selection */
  -webkit-user-select: none;
}
```
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
```
- Call `e.preventDefault()` in all pointer event handlers.

### Distinguishing Draw vs. Pinch
- Track active pointers in a `Map<pointerId, {x, y}>`.
- `activePointers.size === 1` → drawing mode.
- `activePointers.size === 2` → pinch zoom mode. Abort drawing, release all captures.
- On pointerup: remove from map. When map empties → reset to 'none'.

### Coordinate Mapping
```js
function canvasCoordsToGrid(clientX, clientY) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  let x = (clientX - rect.left) * scaleX;
  let y = (clientY - rect.top) * scaleY;
  // Apply inverse viewport transform (zoom/pan)
  x = (x - panX) / zoom;
  y = (y - panY) / zoom;
  return { x: Math.floor(x), y: Math.floor(y) };
}
```

### Stylus / Pressure
- `e.pressure`: 0–1 (0.5 for mouse buttons pressed, 0 otherwise).
- Use for opacity variation: `alpha = pressure` → draw semi-transparent pixel.
- For strict pixel art, ignore pressure; always draw at full opacity, or map pressure to dithering pattern.

### Responsive Canvas Sizing
- Canvas internal resolution = pixel art grid size (e.g. 32x32).
- CSS display size: scale to fit container while preserving aspect ratio.
- High-DPI: multiply backing store by `devicePixelRatio`, scale context so grid coords remain 0..N-1.
- Recalculate on `window.resize` and `orientationchange` (debounced 150ms).

### Touch-Friendly UI
- Minimum touch target: 44x44 CSS pixels.
- Buttons, color swatches, toolbar items: `min-width/min-height: 44px; padding: 8px; touch-action: manipulation`.
- Palette: CSS Grid with `repeat(auto-fill, minmax(44px, 1fr))`.
- Use larger icons and text. Floating/collapsible toolbars on small screens.

---

## 7. Undo / Redo

### Strategy: Full ImageData Snapshots
- Store complete `ImageData` copies before each drawing action.
- For typical pixel art (16x16 to 256x256): 256x256 ImageData = ~256 KB. 100 snapshots = ~25 MB — acceptable.
- Avoids complexity of compressed diffs. Guarantees exact restoration.

### `UndoManager` Class Design
- `undoStack`: array of `ImageData` (states BEFORE each action).
- `redoStack`: array of `ImageData` (states captured when undo is invoked).
- `maxDepth`: configurable (default 100). Circular buffer — shift oldest when exceeded.
- `activeOperation`: boolean — prevents undo/redo mid-stroke.

### Stroke Grouping (One Undo Step per Stroke)
- `beginOperation(ctx, w, h)`: called on `pointerdown`. Captures current canvas state → pushes to `undoStack`. Clears `redoStack`. Sets `activeOperation = true`.
- During stroke (`pointermove`): modify pixels directly on canvas. No snapshots taken.
- `endOperation()`: called on `pointerup`. Sets `activeOperation = false`.

### Undo
```js
undo(ctx, w, h) {
  if (undoStack.length <= 1 || activeOperation) return false; // can't undo past baseline
  const current = ctx.getImageData(0, 0, w, h);
  redoStack.push(current);              // save for redo
  const previous = undoStack.pop();     // get pre-action state
  ctx.putImageData(previous, 0, 0);    // restore
  return true;
}
```

### Redo
```js
redo(ctx, w, h) {
  if (redoStack.length === 0 || activeOperation) return false;
  const current = ctx.getImageData(0, 0, w, h);
  undoStack.push(current);
  const next = redoStack.pop();
  ctx.putImageData(next, 0, 0);
  return true;
}
```

### Keyboard Shortcuts
- Ctrl+Z → undo
- Ctrl+Shift+Z / Ctrl+Y → redo

### Frame Animation Integration
- Each frame gets its own `UndoManager` instance.
- Keyboard shortcuts delegate to the active frame's manager.
- Switching frames ends any active stroke (calls `endOperation`).

### Initialize
```js
undoManager.initialize(ctx, w, h);
// Sets baseline: undoStack = [ctx.getImageData(0, 0, w, h)]
// This baseline is never popped — ensures first action is undoable.
```

### Optional: Coalescing Rapid Clicks
- If the same tool is used within 300ms of the last `endOperation`, skip `beginOperation` snapshot push → new clicks merge into previous undo step.
- Useful for single-pixel placement (pencil tool without dragging).
- Not needed for v1; stroke grouping alone covers 95% of use cases.

---

## Appendix: Key Classes / Structures

### PaletteManager
```js
class PaletteManager {
  constructor(swatchSize = 20, gap = 2) { ... }
  loadPalette(name) { ... }           // switch to preset
  addColor(hex) { ... }               // add custom color
  removeColor(index) { ... }
  handleClick(clientX, clientY, canvasRect) → colorIndex
  renderToContext(ctx, x, y, width, height) { ... }  // draw to display
  save() / load()                     // localStorage
}
```

### UndoManager
```js
class UndoManager {
  constructor(maxDepth = 100) { ... }
  initialize(ctx, w, h) { ... }       // set baseline
  beginOperation(ctx, w, h) { ... }   // start stroke
  endOperation() { ... }              // end stroke
  undo(ctx, w, h) → boolean
  redo(ctx, w, h) → boolean
  canUndo() → boolean
  canRedo() → boolean
}
```

### Viewport State
```js
{ scale: 1, offsetX: 0, offsetY: 0 }
```

### Frame State
```js
{
  frames: OffscreenCanvas[],
  currentIndex: number,
  undoManagers: UndoManager[],    // one per frame
  ghostCache: { prev: OffscreenCanvas, next: OffscreenCanvas },
  isPlaying: false,
  fps: 10
}
```

---

## Implementation Order (Recommended)

1. **Core rendering**: source canvas + display canvas with nearest-neighbor scaling
2. **Zoom and pan**: viewport transforms, mouse wheel, drag-to-pan
3. **Drawing tool**: pencil with pointer events, Bresenham lines
4. **Undo/redo**: UndoManager integration with drawing
5. **Color palette**: PaletteManager, presets, palette UI
6. **PNG export/import**: toBlob download, file input import
7. **Symmetry**: mirror drawing modes
8. **Mobile polish**: pinch zoom, responsive sizing, touch-friendly UI
9. **Animation**: frames array, thumbnails, onion skinning, playback
