# MOSS Layout Designer

A standalone, browser-based tool for visually composing **layout prompts** for the [MOSS-Image](https://github.com/OpenMOSS/MOSS-Image) model. Draw, move, and edit bounding boxes on a user-definable canvas, assign content to each box, and export a JSON prompt that MOSS-Image consumes directly in ComfyUI.

- **File:** `index.html` — a single self-contained file. No dependencies, no build step, no server. Double-click to open in any modern browser.
- **No ComfyUI connection required.** The app only produces JSON; either a human or an agent (e.g., Cline via the ComfyUI MCP) feeds that JSON into a workflow.
- Works entirely offline. Projects are saved/loaded as local files.

---

## 1. App functionality

### Canvas

- User-definable canvas size, always constrained to multiples of 16 (MOSS-Image requires this).
- Preset dropdown offering common aspect ratios at MOSS-recommended pixel budgets (~1M / 1.5M / 2M pixels), in both orientations, plus a square 1M default.
- The canvas origin marker (crosshair at center) reflects MOSS's `abs_centralized` coordinate system: **0,0 is the center of the image**, x increases rightward, y increases downward.
- The status bar at the lower-left of the canvas shows live `abs_centralized` coordinates under the cursor while you move the mouse.

### Boxes

Each box becomes one `item` in the exported layout JSON. Boxes support:

| Action | How |
|---|---|
| Create | Drag on empty canvas area |
| Move | Drag inside a box |
| Resize | Drag any of the 8 corner/edge handles |
| Nudge | Arrow keys (Shift+Arrow = 16 px steps) |
| Duplicate | `Ctrl+D` or the Duplicate button |
| Delete | `Del`/`Backspace` or the Delete button |
| Reorder (z/priority) | Forward / Back buttons; list order = front-to-back |
| Undo | `Ctrl+Z` |
| Snap | "Snap 16" checkbox — snaps all coordinates to the 16 px grid |

Boxes are constrained to stay inside the canvas and cannot shrink below 16×16 px.

### Reference image

"Ref image" loads a PNG/JPG (e.g., a previously generated MOSS render) as a translucent underlay on the canvas so you can trace or adjust boxes against a known-good composition. "Clear ref" removes it. The reference is never saved with the project.

### Inspector tab (per selected box)

- **Label (category):** `text` or `object` — the item category MOSS expects.
- **Caption (item description):** a short natural-language description of what belongs in the box.
- **Text to render:** only used when label is `text`. Use line breaks to control rendered line structure.
- **X, Y, W, H:** top-left pixel coordinates and size, editable numerically (snapped to 16).
- **Bounding box readout:** the box's live `bbox` in `abs_centralized` `yxyx` form, exactly as it will be exported.
- **Boxes list:** all boxes in z-order (top of list = front); click to select.

### Scene tab (global fields)

- **summary.caption:** overall scene/style description for the whole image.
- **summary.meta:** comma-separated meta tags (e.g., `poster`), exported as the `meta` array.
- **Extra natural-language prompt (optional):** prose that can be prepended before the JSON — MOSS supports combining natural language and structured input. Use "Copy with prompt" to get it formatted.
- **Negative prompt:** stored with the project only. It is deliberately **not** embedded in the layout JSON — pass it to ComfyUI as a separate conditioning field.

### JSON tab (export + validation)

- Live, pretty-printed layout JSON in the exact MOSS-Image schema (§2).
- **Copy JSON** — copy the layout JSON to the clipboard.
- **Copy with prompt** — copy the optional extra prose prompt followed by the JSON.
- **Download .json** — save the layout JSON as a file.
- **Validation messages** (below the JSON preview):
  - *Error:* canvas dimensions not multiples of 16; a box extends beyond the canvas; no boxes defined.
  - *Warning:* empty `summary.caption`; a `text` item with empty text; an item with an empty caption; overlapping boxes (allowed, but flagged so you can confirm intent).
  - *OK:* layout is valid and ready to send.

### Save / Load

- **Save project** downloads a full project file (`moss_layout_project_<timestamp>.json`, schema in §3) containing canvas size, global fields, and all boxes.
- **Load / import** accepts either:
  1. A **project file** saved by this app (full round-trip restore), or
  2. A **MOSS layout JSON** in the model's native format (§2) — boxes are converted back onto the canvas for visual editing.


---

## 2. MOSS layout-prompt JSON schema (as exported)

This is the format MOSS-Image-Base consumes, per its official `USERGUIDE.md` ("Layout Prompts" section).

```json
{
  "summary": {
    "meta": ["poster"],
    "canvas": "single",
    "caption": "A minimalist poster with a cream background"
  },
  "items": [
    {
      "label": "text",
      "caption": "Heading",
      "text": "Built with Cline!",
      "bbox": [-448, -384, -320, 384]
    },
    {
      "label": "object",
      "caption": "A cartoon illustration of a happy bee flying near a colorful flower",
      "bbox": [-176, -192, 208, 192]
    }
  ],
  "bbox_coord_space": "abs_centralized",
  "bbox_format": "yxyx"
}
```

### 2.1 Field definitions

| Field | Meaning |
|---|---|
| `summary.caption` | Description of the overall scene and style. |
| `summary.meta` | Array of style/content meta tags (e.g., `"poster"`). |
| `summary.canvas` | `"single"` — a single image canvas. |
| `items[].label` | Item category. The app supports `text` and `object`. |
| `items[].caption` | Short description of what belongs in that box. |
| `items[].text` | The literal text to render. **Present only for `label: "text"` items.** |
| `items[].bbox` | The bounding box, ordered `[y_min, x_min, y_max, x_max]`. |
| `bbox_format` | Always `"yxyx"` — coordinates ordered `[y_min, x_min, y_max, x_max]`. |
| `bbox_coord_space` | Always `"abs_centralized"` — absolute pixels, origin at canvas center. |

### 2.2 Coordinate system: `abs_centralized`

- Coordinates are **absolute pixels**, not normalized.
- **The origin is the center of the canvas**: x grows to the right, y grows downward.
- A 1024×1024 canvas spans **−512…+512** on both axes. A W×H canvas spans −W/2…+W/2 and −H/2…+H/2.
- Conversion from ordinary top-left-origin pixel coordinates: `x_c = x − W/2`, `y_c = y − H/2`.
- The bbox exported from a box at top-left `(x, y)` with size `(w, h)` on a W×H canvas is:

```
bbox = [y − H/2,               # y_min
        x − W/2,               # x_min
        y + h − H/2,           # y_max
        x + w − W/2]           # x_max
```

> MOSS chose absolute pixel coordinates deliberately, so a given layout maps to specific physical positions at any resolution/aspect ratio. **This means a layout is resolution-specific**: generating a 1024×1024 layout at 1368×768 would put every box in the wrong place. Always match the workflow's latent dimensions to the canvas the layout was drawn on (the app displays the canvas size, and the project file stores it).

### 2.3 Text items

When `label` is `"text"`, include the `text` field with the exact string to render. Line breaks (`\n`) inside `text` control line wrapping. The caption describes *how* the text should look (size, alignment, color, placement).

### 2.4 Combining with natural language

MOSS-Image supports mixing prose and structured input. Optionally prepend a natural-language description before the JSON (the app's "Copy with prompt" does this using the Scene tab's "Extra natural-language prompt" field).

### 2.5 Relevant MOSS-Image inference settings (reference)

- **Resolution:** 1M / 1.5M / 2M total pixels recommended (the model extrapolates 0.25M–4M). Dimensions must be multiples of 16.
- **Aspect ratio:** arbitrary.
- **Steps:** ≥ 50 recommended for the base model (the 8-step DMD turbo LoRA skips this).
- **CFG:** default 3.0 (range 3.0–6.0); increase for short prompts.
- **Shift:** 5.0; **Sampler:** Euler recommended.
- **SLG:** optional quality enhancement (see USERGUIDE).

---

## 3. Project file schema (app-specific)

The file saved by **Save project** is *not* the model's layout format — it is the app's own round-trip format:

```json
{
  "app": "moss-layout-designer",
  "version": 1,
  "canvas": { "w": 1024, "h": 1024 },
  "meta": "poster",
  "summaryCaption": "A minimalist poster with a cream background",
  "extraPrompt": "",
  "negativePrompt": "",
  "boxes": [
    {
      "id": "baumezne",
      "label": "text",
      "caption": "Heading",
      "text": "Built with Cline!",
      "x": 128, "y": 64, "w": 768, "h": 128
    },
    {
      "id": "brjudxp5",
      "label": "object",
      "caption": "A cartoon illustration of a happy bee flying near a colorful flower",
      "text": "",
      "x": 320, "y": 336, "w": 384, "h": 384
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `canvas.w` / `canvas.h` | Canvas size in px (multiples of 16). |
| `meta` | Comma-separated meta tags (becomes `summary.meta`). |
| `summaryCaption` | Becomes `summary.caption`. |
| `extraPrompt` | Optional prose prompt (not part of the layout JSON itself). |
| `negativePrompt` | Negative prompt; pass separately in ComfyUI. |
| `boxes[].id` | Internal box id (does not appear in the exported layout). |
| `boxes[].label` | `text` or `object`. |
| `boxes[].caption` | Item description. |
| `boxes[].text` | Rendered text (`text` items only; ignored for `object`). |
| `boxes[].x/y/w/h` | **Top-left pixel coordinates and size** (canvas origin at top-left). The export step converts these to `abs_centralized` yxyx. |

---

## 4. Sending the layout to ComfyUI

The app does not talk to ComfyUI. The exported JSON goes into a workflow via a **MOSS-Image Layout Prompt** node (ComfyUI core: `model/conditioning/moss_image`) as its `prompt` text input — i.e., the JSON string is passed as text, exactly as written above.

### Resolution-matching checklist

1. Canvas size in the app == latent image size in the workflow (both multiples of 16).
2. If the workflow uses a `ResolutionSelector`, confirm it produces the same W×H as the app canvas.
3. The layout JSON's `bbox` values are absolute pixels for that canvas — do not reuse a layout across resolutions without re-drawing.

### Filename-prefix note (verified 2026-10-28)

The `%date:yyyyMMdd_hhmmss%` token in a `SaveImage` node's `filename_prefix` **does not expand in API-mode submissions** — it produces a 0-byte file whose PNG payload ends up in an NTFS alternate data stream named after the literal token, and the saved filename in the history is also wrong. Workaround for agent-driven runs: **replace the prefix with a literal timestamp** (e.g., `20261028_175427_TestMoss`) before submitting. The same token works fine when run from the ComfyUI UI, since the UI expands it client-side.

### Proven end-to-end pipeline (2026-10-28)

1. Draw layout in the app → **Save project**.
2. Agent reads the project JSON, converts to `abs_centralized` yxyx, and injects the JSON string into the workflow's prompt field.
3. Submit to the local ComfyUI API (`http://127.0.0.1:8188/prompt`) and poll `/history/<prompt_id>`.
4. Verify the PNG landed in the configured output folder with a literal timestamp filename.

---

## 5. Tips

- The app opens with the USERGUIDE's "EXPLORE" poster example as a demo — its exported coordinates reproduce the documented example exactly, which serves as a sanity check that the coordinate conversion is correct.
- Use a previously generated image as the reference underlay to iterate on a composition: load it, adjust boxes, re-export.
- Keep `summary.caption` consistent with the boxes (e.g., if a caption says "title at the top," put a text box at the top).
- MOSS-Image's layout prompting is experimental and benefits from the model's own "layout enhancer" approach — an LLM (or Cline) can generate layout JSONs directly from a content list; load them into the app via "Load / import" to adjust visually.

