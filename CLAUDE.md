# Scrollyteller

A no-code, browser-based editor where students build scrollytelling stories on a vertical, video-editor-style timeline and export a standalone HTML file.

## Who it's for

The users are a math teacher and their students, doing a descriptive-statistics project. The students are not advanced, so keep the UI plain-language and forgiving: no jargon, undo everywhere, and helpful hints instead of errors. They will mostly be on Chromebooks or laptops in Chrome.

## Hard constraints

- **The whole app is one file: `index.html`.** There is no build step, no npm dependencies, no external CDN scripts or fonts, and no server. It must work when opened from `file://` and when hosted on **GitHub Pages**, where it's served from the repo root as `index.html`.
- **Exported stories must be fully self-contained.** That means one `.html` file with images embedded as data URLs and no network requests.
- Keep it simple. Prefer a small, obvious feature over a flexible one.

## Architecture (all in `index.html`)

- **`scrollyRuntime()`**: the rendering engine, shared by the editor preview **and** the exported story. Its source is copied verbatim into every export via `scrollyRuntime.toString()`, so it **must stay self-contained**: no references to outside variables, and ES5-ish code. It exposes three functions:
  - `mount(stage, project)`: builds the DOM for each clip.
  - `update(mounted, t)`: sets visibility, position, effects and the Ken Burns transform for scroll position `t`.
  - `length(project)`: the story length in screens.
- **`RUNTIME_CSS`**: CSS shared by the editor and the exports. It's injected into the editor at boot and written into exports.
- **`buildExport()`**: produces the exported HTML. The project JSON is embedded in `<script type="application/json" id="scrolly-data">`, with `<` escaped as `<`. "Open…" can read that block back, so an exported story can be reopened for editing. Inside template literals, write `<\/script>`.
- **Editor state** lives in globals: `project`, `sel` (the selected clip id), `playhead` (in screens) and `pxPer` (timeline zoom).
  - `render()` redraws the timeline and the preview.
  - `renderInspector()` rebuilds the right-hand panel.
  - `syncInspector()` updates field values without rebuilding, so typing isn't interrupted.
  - The preview only remounts when `mountKey` changes (clip ids, text or image src), so dragging stays cheap.
- **History**:
  - `commit()` pushes the previous JSON snapshot onto `undoStack` and auto-saves to `localStorage` under `scrollyteller-project`.
  - Live edits happen on `input` events and are committed on `change` or `pointerup`.
- **Inspector fields** are bound declaratively with `data-k="path.to.key"`; add `data-scope="project"` for project-level settings. `applyField` handles all of them.

### Units and data model

- **Time** is measured in "screens" (1 = one viewport height of scrolling). Clip `start` and `end` snap to 0.1.
- **Position and size** (`x`, `y`, `w`, `h`) are percentages of the stage. Text has auto height, so `h` only applies to images.
- **Text clips store rich text in `html`.**
  - It's edited in the `#rt` contenteditable using `document.execCommand`, with a toolbar for bold, italic, underline, bullet lists, links, colour and clear formatting.
  - It's always passed through `sanitize()`, which keeps simple tags, inline `color` styles and http(s)/mailto links (opened with `target=_blank`), and strips everything else.
  - `newClip()` sanitises on every load and converts old plain-text `text` fields.
  - The runtime trusts `html` and sets it with `innerHTML`, so anything that writes `html` must sanitise first.
- **Font size** is `size / 10` in `cqmin`. The stage is a size container, so the preview and the export scale identically.
- **Effects**:
  - `enter` and `exit` each take one of `none | fade | left | right | top | bottom | zoom`, with a duration in screens (`enterLen` / `exitLen`).
  - `motion` is `none | drift`.
- **Image camera** is `kb: {z0, x0, y0, z1, x1, y1}`.
  - It's interpolated over the clip's whole duration.
  - Pan runs from -100 to 100 as a fraction of the pannable range at the current zoom, so the image's edges never show.
- **Layers**: lane index = z-order, so higher lanes draw on top.
- **Loading**: `normalize()` and `newClip()` fill in defaults whenever a project is loaded. **When adding a field, give it a default in `newClip()`** so older saved projects keep working.

## Testing

There are no automated tests in the repo. To verify a change:

1. Extract the inline `<script>` and run `node --check` on it.
2. Drive it in headless Chromium using `playwright-core`, installed in a temp dir rather than the repo:
   - Load `index.html`.
   - Interact with `.clip`, `.h-bot` and the other controls.
   - Call `buildExport()` in the page and save the result.
   - Open the export, `scrollTo` several `t * innerHeight` positions, and take screenshots.
   - Check for page errors.
3. Check the export at both laptop (1280×720) and phone (390×844) sizes.

## Status and ideas for later

Done:
- The timeline: drag, stretch, change layer, scrub.
- Text and image clips, with drag-and-drop and paste for images.
- Rich text in text clips, including links.
- Effects and the image camera.
- Dragging and resizing in the preview.
- Undo/redo, auto-save, saving and opening project JSON, and reopening exported HTML.
- Preview in a new tab and export.
- A demo story about hours of sleep (mean, median, outlier, range vs. IQR).

Possible next steps (not started):
- A built-in chart clip, such as a histogram, bar chart or box plot made from typed-in numbers.
- Phone-specific layouts.
- Hosting on GitHub Pages: the repo isn't created yet.
