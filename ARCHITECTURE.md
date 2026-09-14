# Portabooth — Architecture & Context

Deep-dive reference for future sessions/agents working on this repo. Read this before making changes so you don't have to re-derive context from scratch.

## What this project is

A browser-based photobooth ("Snapstrip"). User taps a button, the browser's webcam captures 4 photos on a countdown timer, and the shots are composited into a single vertical photo-strip image (like a real photobooth strip) that can be saved, shared, or printed.

**Everything happens client-side.** No backend, no server, no build step, no dependencies. The camera feed and captured images never leave the device — there is nothing to upload to.

## Current state

- `photobooth.html` — the entire application: markup, CSS, and JS in one file (~530 lines).
- `qrcode.min.js` — vendored third-party QR code generator (davidshimjs/qrcodejs, MIT), loaded via a local `<script src>` tag. See "Custom template sharing" below for why it's vendored instead of a CDN `<script>` tag.
- `README.md` — one-line project blurb.
- No package.json, no framework, no bundler. Opening the HTML file in a browser (or serving it statically) is the whole deploy story.

Deliberate choice, not a placeholder: [conversation with the project owner](.) concluded plain HTML/vanilla JS is the right call here over React — the app is a single screen with no routing, state is a handful of DOM refs + one `shots` array, and it uses browser APIs (`getUserMedia`, `<canvas>`, Web Share) directly, which a framework wouldn't simplify. A framework would only earn its cost if this grows multiple screens/routes or shared state across views. See "If this grows" below.

## Architecture of `photobooth.html`

### Layout (HTML)
- `#templateNote` — one-line "Using a shared custom template" indicator, shown only when the page was opened via a link/QR that encodes a saved template (see below).
- `.customize` (`<details>`) — the template editor: frame-theme `<select>`, caption `<input>`, show-date checkbox, and the "Get shareable link" flow (link `<input readonly>` + copy button + `#qrOutput` QR code container). Collapsed by default so the primary shoot flow stays uncluttered.
- `.stage` — the live camera view: `<video>` element (mirrored front camera feed), flash overlay, countdown overlay, shot counter (`0 / 4` etc.), and the Start/Retake buttons.
- `#permHint` — camera-permission helper text shown before capture / on `getUserMedia` failure.
- `.strip-wrap` — the result screen: rendered photo strip (`<canvas id="stripCanvas">`) plus Save / Share / Print / New-strip buttons. Hidden until a session completes. The caption/date text is drawn directly onto this canvas (not a separate DOM overlay) so it's included in Save/Share/Print output — see "Custom template sharing" below.
- `#workCanvas` — declared but currently **unused / dead** (see Known issues below).

### Styling
Single `<style>` block, CSS custom properties for the "velvet curtain photobooth" theme (`--curtain`, `--bulb`, `--paper`, `--brass`, etc.), defined on `:root` and overridden per theme via `:root[data-theme="noir|blush|mint"]`. Includes a `@media print` rule so "Print" only prints the strip, not the whole page chrome.

### JS (single `<script>` block)
No modules, no classes — a flat script with DOM refs at the top and a handful of functions:

| Function | Responsibility |
|---|---|
| `initCamera()` | Requests `getUserMedia` (front camera, ideal 1280×960), binds stream to `<video>`. Shows `permHint` text on failure. Runs immediately on page load. |
| `runCountdown(seconds)` | Async, updates `#countdown` overlay text once per second via `sleep()`. |
| `doFlash()` | CSS-opacity flash effect on `.flash` overlay, triggered right before each capture. |
| `captureFrame()` | Grabs one frame from `<video>` onto an off-DOM `<canvas>`, cover-fit-cropped to a fixed 500×375 (4:3) cell, mirrored to match the on-screen preview. Returns the canvas. |
| `buildStrip()` | Stacks the captured canvases vertically (with padding/gaps) onto the visible `#stripCanvas`, reading `--paper`/`--ink` from the active theme, then draws the caption/date text onto the canvas itself. |
| `runSession()` | Orchestrates one full run: resets `shots`, loops 4× doing countdown → flash → capture, then calls `buildStrip()` and reveals the result screen. |
| Save/Share/Print handlers | `saveBtn` → `canvas.toDataURL('image/png')` download link. `shareBtn` → `navigator.share` with a `File` (only shown if `navigator.canShare` exists). `printBtn` → `window.print()`. `againBtn` → hides the result screen to shoot again. |
| `applyTheme(theme)` / `applyConfig(cfg)` | Sets/removes `data-theme` on `<html>` and syncs the customize form fields to a `config` object. |
| `encodeConfig(cfg)` / `decodeConfig(str)` / `readConfigFromURL()` | Pack/unpack `{theme, caption, showDate}` to/from a base64-JSON `#c=` URL hash. `decodeConfig` validates `theme` against the known list and clamps `caption` length — untrusted input, since it comes from a URL someone else generated. |
| `onConfigFieldChange()` | Fired on customize-field input/change; updates `config`, re-applies the theme, and re-runs `buildStrip()` live if a strip was already captured. |
| `renderQRCode(container, text)` | Renders a QR code via the vendored `QRCode` global, retrying increasing QR "type" (matrix size) until the payload fits — the library doesn't auto-size and throws on overflow otherwise. |

**State** is minimal and intentionally not framework-managed: `stream` (MediaStream), `shots` (array of captured `<canvas>` elements), and `config` (`{theme, caption, showDate}`), all module-level `let` variables closed over by the functions above.

**Constants** controlling capture geometry: `SHOT_COUNT = 4`, `FRAME_W/FRAME_H = 500×375` (4:3), `PADDING = 20`, `GAP = 14`, `CAPTION_H = 46`.

### Custom template sharing

Feature: a user can set a frame theme, caption, and date visibility, then get a link/QR code that reproduces that exact template for whoever opens it — no login, no per-user frame upload.

Design choice (discussed with the project owner before building): **entirely static, config packed into the URL** rather than adding any backend/storage. Concretely: `{theme, caption, showDate}` → `JSON.stringify` → UTF-8-safe base64 → `#c=<blob>` URL hash → QR code generated client-side from that URL via the vendored `qrcode.min.js`. Opening the link re-decodes the hash and calls `applyConfig()` before the user does anything else.

This was chosen over two heavier alternatives (still on the table if requirements grow):
- **Full accounts** — login + per-user frame management. Rejected as overkill for "share one template with someone."
- **Minimal serverless storage** — real image upload, stored server-side under a short ID. Would be needed if arbitrary custom frame *graphics* (not just theme/caption) become a requirement, since an uploaded image can't reasonably live inside a URL/QR code. This is the natural next step if that's ever wanted — see the conversation history around 2026-09-14 for the fuller tradeoff writeup.

Because of this choice, current "frames" are limited to the built-in `THEMES` list (`classic`, `noir`, `blush`, `mint`) plus free-text caption and a date toggle — not arbitrary uploaded graphics. Keep the encoded payload small (short strings/enums only) so the QR code stays scannable.

`qrcode.min.js` is vendored (copied into the repo) rather than loaded from a CDN `<script src>` specifically to preserve the app's "everything runs on your device, no network dependency beyond the camera" property — a CDN script would make template rendering depend on a third party being up.

### Mobile-specific details already handled
- `playsinline muted autoplay` on `<video>` — required for iOS Safari to render the camera feed inline instead of forcing fullscreen.
- `viewport` meta tag for correct scaling.
- `facingMode: 'user'` requests the front camera by default.

## Known issues / dead code (as of 2026-09-14)

- `#retakeBtn` — rendered in HTML, referenced in JS (`retakeBtn.style.display = 'none'` in `runSession`), but **no click handler is ever attached and it's never shown** (`display:none` is its only state). Effectively vestigial. If "retake mid-session" is a wanted feature, this is the hook to wire up; otherwise it can be deleted.
- `#workCanvas` — declared, grabbed via `getElementById`, never read or drawn to anywhere else in the script. Looks like leftover scaffolding from an earlier compositing approach. Safe to remove unless a future feature needs a scratch canvas.

## If this grows

Reach for a framework/build step only if the project actually grows beyond a single screen — e.g. a gallery of past strips, multiple routes, or shared state across views. (A settings/theme picker alone did *not* require this — see "Custom template sharing" above, done with plain URL-encoded state.) A lighter first step than full React would be Vite + vanilla TS to get a dev server and type-checking without adopting a component framework. Until then, keep it a single static file (plus small vendored dependencies where justified) — it keeps load time minimal, which matters more than usual here since it's a mobile-first camera app.

If arbitrary custom frame *image* uploads become a requirement, that's the point where minimal serverless storage (no login, just an upload endpoint + object storage + short-ID lookup) becomes the right next step — see "Custom template sharing" above for the fuller tradeoff.

## Working conventions for this repo

- Keep it a single static HTML file unless there's a concrete reason to split it (see "If this grows").
- No build step should be introduced without discussing it first — the zero-dependency, drop-in-a-static-host nature of this repo is a deliberate feature, not an oversight.
- When editing capture/compositing logic (`captureFrame`, `buildStrip`), test on an actual mobile browser (iOS Safari in particular) — front-camera aspect ratios and `playsinline` behavior vary more there than on desktop webcams.
