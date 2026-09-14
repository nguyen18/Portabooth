# Portabooth — Architecture & Context

Deep-dive reference for future sessions/agents working on this repo. Read this before making changes so you don't have to re-derive context from scratch.

## What this project is

A browser-based photobooth ("Snapstrip"). User taps a button, the browser's webcam captures 4 photos on a countdown timer, and the shots are composited into a single vertical photo-strip image (like a real photobooth strip) that can be saved, shared, or printed.

**Everything happens client-side.** No backend, no server, no build step, no dependencies. The camera feed and captured images never leave the device — there is nothing to upload to.

## Current state

- `photobooth.html` — the entire application: markup, CSS, and JS in one file (~440 lines).
- `README.md` — one-line project blurb.
- No package.json, no framework, no bundler. Opening the HTML file in a browser (or serving it statically) is the whole deploy story.

Deliberate choice, not a placeholder: [conversation with the project owner](.) concluded plain HTML/vanilla JS is the right call here over React — the app is a single screen with no routing, state is a handful of DOM refs + one `shots` array, and it uses browser APIs (`getUserMedia`, `<canvas>`, Web Share) directly, which a framework wouldn't simplify. A framework would only earn its cost if this grows multiple screens/routes or shared state across views. See "If this grows" below.

## Architecture of `photobooth.html`

### Layout (HTML, lines ~245-281)
- `.stage` — the live camera view: `<video>` element (mirrored front camera feed), flash overlay, countdown overlay, shot counter (`0 / 4` etc.), and the Start/Retake buttons.
- `#permHint` — camera-permission helper text shown before capture / on `getUserMedia` failure.
- `.strip-wrap` — the result screen: rendered photo strip (`<canvas id="stripCanvas">`) plus Save / Share / Print / New-strip buttons. Hidden until a session completes.
- `#workCanvas` — declared but currently **unused / dead** (see Known issues below).

### Styling
Single `<style>` block, CSS custom properties for the "velvet curtain photobooth" theme (`--curtain`, `--bulb`, `--paper`, `--brass`, etc.). Includes a `@media print` rule so "Print" only prints the strip, not the whole page chrome.

### JS (single `<script>` block, lines ~282-440)
No modules, no classes — a flat script with DOM refs at the top and a handful of functions:

| Function | Responsibility |
|---|---|
| `initCamera()` | Requests `getUserMedia` (front camera, ideal 1280×960), binds stream to `<video>`. Shows `permHint` text on failure. Runs immediately on page load. |
| `runCountdown(seconds)` | Async, updates `#countdown` overlay text once per second via `sleep()`. |
| `doFlash()` | CSS-opacity flash effect on `.flash` overlay, triggered right before each capture. |
| `captureFrame()` | Grabs one frame from `<video>` onto an off-DOM `<canvas>`, cover-fit-cropped to a fixed 500×375 (4:3) cell, mirrored to match the on-screen preview. Returns the canvas. |
| `buildStrip()` | Stacks the 4 captured canvases vertically (with padding/gaps) onto the visible `#stripCanvas`. |
| `runSession()` | Orchestrates one full run: resets `shots`, loops 4× doing countdown → flash → capture, then calls `buildStrip()` and reveals the result screen. |
| Save/Share/Print handlers | `saveBtn` → `canvas.toDataURL('image/png')` download link. `shareBtn` → `navigator.share` with a `File` (only shown if `navigator.canShare` exists). `printBtn` → `window.print()`. `againBtn` → hides the result screen to shoot again. |

**State** is minimal and intentionally not framework-managed: `stream` (MediaStream) and `shots` (array of captured `<canvas>` elements), both module-level `let` variables closed over by the functions above.

**Constants** controlling capture geometry: `SHOT_COUNT = 4`, `FRAME_W/FRAME_H = 500×375` (4:3), `PADDING = 20`, `GAP = 14`.

### Mobile-specific details already handled
- `playsinline muted autoplay` on `<video>` — required for iOS Safari to render the camera feed inline instead of forcing fullscreen.
- `viewport` meta tag for correct scaling.
- `facingMode: 'user'` requests the front camera by default.

## Known issues / dead code (as of 2026-09-14)

- `#retakeBtn` — rendered in HTML, referenced in JS (`retakeBtn.style.display = 'none'` in `runSession`), but **no click handler is ever attached and it's never shown** (`display:none` is its only state). Effectively vestigial. If "retake mid-session" is a wanted feature, this is the hook to wire up; otherwise it can be deleted.
- `#workCanvas` — declared, grabbed via `getElementById`, never read or drawn to anywhere else in the script. Looks like leftover scaffolding from an earlier compositing approach. Safe to remove unless a future feature needs a scratch canvas.

## If this grows

Reach for a framework/build step only if the project actually grows beyond a single screen — e.g. a gallery of past strips, settings/theme picker, multiple routes, or shared state across views. A lighter first step than full React would be Vite + vanilla TS to get a dev server and type-checking without adopting a component framework. Until then, keep it a single static file — it keeps load time minimal, which matters more than usual here since it's a mobile-first camera app.

## Working conventions for this repo

- Keep it a single static HTML file unless there's a concrete reason to split it (see "If this grows").
- No build step should be introduced without discussing it first — the zero-dependency, drop-in-a-static-host nature of this repo is a deliberate feature, not an oversight.
- When editing capture/compositing logic (`captureFrame`, `buildStrip`), test on an actual mobile browser (iOS Safari in particular) — front-camera aspect ratios and `playsinline` behavior vary more there than on desktop webcams.
