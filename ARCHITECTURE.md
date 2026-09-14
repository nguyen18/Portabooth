# Portabooth — Architecture & Context

Deep-dive reference for future sessions/agents working on this repo. Read this before making changes so you don't have to re-derive context from scratch.

## What this project is

A browser-based photobooth ("Portabooth" — formerly named "Snapstrip" in-app; renamed 2026-09-14 to match the repo/product name). User taps a button, the browser's webcam captures 4 photos on a countdown timer, and the shots are composited into a single vertical photo-strip image (like a real photobooth strip) that can be saved, shared, or printed.

**Everything happens client-side.** No backend, no server, no build step, no dependencies. The camera feed and captured images never leave the device — there is nothing to upload to.

## Current state

- `index.html` — the entire application: markup, CSS, and JS in one file (~590 lines). Named `index.html` (renamed from `photobooth.html` 2026-09-14) so GitHub Pages serves it as the site root.
- `CNAME` — custom domain (`portabooth.studio`) for GitHub Pages.
- `qrcode.min.js` — vendored third-party QR code generator (davidshimjs/qrcodejs, MIT), loaded via a local `<script src>` tag. See "Custom template sharing" below for why it's vendored instead of a CDN `<script>` tag.
- `fonts/Unna-{Regular,Bold,Italic,BoldItalic}.ttf` — vendored Unna typeface (SIL Open Font License, `fonts/Unna-OFL.txt`), used for the caption/date drawn onto the exported strip. Vendored for the same reason as the QR library: no third-party dependency at runtime.
- `sounds/shutter.mp3` — vendored camera-shutter sound effect (user-supplied download, ~1s, no license file included alongside it — unlike the font/QR library, there's no bundled license text to point to if this is ever redistributed beyond this repo), played on each shot.
- `sounds/lofi-loop.mp3` — vendored background music loop (user-supplied download, ~22s, same no-license-file caveat as the shutter sound), looped on every page — see "Background music" below.
- `README.md` — one-line project blurb.
- No package.json, no framework, no bundler. Opening the HTML file in a browser (or serving it statically) is the whole deploy story.
- Deployed via GitHub Pages, custom domain `portabooth.studio` (DNS managed in Squarespace Domains, pointed at GitHub's Pages IPs/host). Pushing to `main` redeploys automatically — no CI step needed.

Deliberate choice, not a placeholder: [conversation with the project owner](.) concluded plain HTML/vanilla JS is the right call here over React — the app is a single screen with no routing, state is a handful of DOM refs + one `shots` array, and it uses browser APIs (`getUserMedia`, `<canvas>`, Web Share) directly, which a framework wouldn't simplify. A framework would only earn its cost if this grows multiple screens/routes or shared state across views. See "If this grows" below.

## Architecture of `index.html`

### Layout (HTML)
- `#templateNote` — one-line "Using a shared custom template" indicator, shown only when the page was opened via a link/QR that encodes a saved template (see below).
- `#bgMusic` (`<audio loop>`) / `#musicToggle` — background music loop and its on/off toggle button, shown/active on every page (main page and shared template links alike) — see "Background music" below.
- `.customize` (`<details open>`) — the template editor: site-theme `<select>`, caption `<input>`, Bold/Italic style checkboxes, show-date checkbox, a live `#captionPreview` canvas, and the "Get shareable link" flow (link `<input readonly>` + copy button + `#qrOutput` QR code container). Expanded by default (`open` attribute) so a first-time visitor lands on customization, not the camera — still collapsible via the `<summary>`. Hidden entirely for shared-link recipients regardless of `open`, via `customizeDetails.style.display = 'none'` (see "Custom template sharing" below).
- `.stage` — the live camera view: `<video>` element (mirrored front camera feed), flash overlay, countdown overlay, shot counter (`0 / 4` etc.), and the Start/Retake buttons.
- `#permHint` — camera-permission helper text shown before capture / on `getUserMedia` failure.
- `.strip-wrap` — the result screen: rendered photo strip (`<canvas id="stripCanvas">`), a `#dateStyleField` date-format `<select>` (`'long'` / `'numeric'` / `'off'` — only visible once a strip exists and `config.showDate` is true; `'off'` hides the date without touching the locked `config.showDate`, see "Recipient lockdown" below), and the Save / Share / Print / New-strip buttons. Hidden until a session completes. The caption/date text and the cut-guide dashes are drawn directly onto the canvas (not a separate DOM overlay) so they're included in Save/Share/Print output — see "Custom template sharing" below.
- `#workCanvas` — declared but currently **unused / dead** (see Known issues below).

### Styling
Single `<style>` block. Two independent color systems, which was a point of real confusion during development and is worth keeping straight:
- **Site theme** — CSS custom properties for the "velvet curtain photobooth" *page chrome* (`--curtain`, `--bulb`, `--brass`, etc.), defined on `:root` and overridden via `:root[data-theme="noir|blush|mint"]`. Controlled by the "Site theme" select. Does **not** affect the exported strip.
- **Strip appearance** — always plain white paper with dark ink (`STRIP_PAPER`/`STRIP_INK` constants in JS), regardless of site theme, so a printed strip always looks like a real photobooth strip. `.strip-frame` background is hardcoded `#ffffff` for the same reason.

Four `@font-face` declarations load the vendored Unna weights/styles (regular/bold/italic/bold-italic) for the canvas-drawn caption. Includes a `@media print` rule so "Print" only prints the strip, not the whole page chrome.

### JS (single `<script>` block)
No modules, no classes — a flat script with DOM refs at the top and a handful of functions:

| Function | Responsibility |
|---|---|
| `initCamera()` | Calls `startCamera(facingMode)` (starts on the front camera), then `enumerateDevices()` to check for more than one `videoinput` — `#flipCameraBtn` only becomes visible if a second camera actually exists (most laptops only have one). Runs immediately on page load. |
| `startCamera(mode)` | Requests `getUserMedia` with the given `facingMode` (ideal 1280×960). Only stops the previous stream's tracks and commits `facingMode`/`video.classList('mirrored')` once the new stream has actually succeeded — a failed camera switch leaves the working camera running rather than blanking the viewfinder. Shows `permHint` text on failure. |
| `runCountdown(seconds)` | Async, updates `#countdown` overlay text once per second via `sleep()`. |
| `doFlash()` | CSS-opacity flash effect on `.flash` overlay plus `playShutterSound()`, triggered right before each capture. Uses a forced reflow (`void flashEl.offsetHeight`) between the "flash on" and fade-out style writes — without it, the two changes could get coalesced into a single paint and the flash would inconsistently fail to render. |
| `playShutterSound()` | Plays the vendored `sounds/shutter.mp3` via the shared `shutterAudio` (`HTMLAudioElement`), resetting `currentTime` first so rapid repeats restart cleanly. |
| `captureFrame()` | Grabs one frame from `<video>` onto an off-DOM `<canvas>`, cover-fit-cropped to a fixed 500×375 (4:3) cell. Mirrored to match the on-screen preview only when `facingMode === 'user'` — the back camera captures un-mirrored, like a real camera. Returns the canvas. |
| `buildStrip()` | Async. Stacks the captured canvases vertically (with padding/gaps) onto `#stripCanvas` against a fixed white background, then calls `drawCaptionLines()` for the caption band and draws the dashed cut-guide border around the full canvas edge. |
| `renderCaptionPreview()` | Async. Draws the same caption/date band (via `computeCaptionLayout()` + `drawCaptionLines()`) onto the small `#captionPreview` canvas in the customize panel, at the same pixel width as the real strip so it's an exact preview, not an approximation. Called from `applyConfig()` and `onConfigFieldChange()` so it updates live as the user types/toggles, before any photo is taken. |
| `computeCaptionLayout()` / `drawCaptionLines(ctx, width, top, layout)` | Shared caption logic used by both `buildStrip()` and `renderCaptionPreview()`, so the live preview and the final export can never drift out of sync. `computeCaptionLayout()` returns `{captionLine, dateLine, captionH}`; `drawCaptionLines()` draws those two lines at a given vertical offset. |
| `captionFont(size)` | Builds a canvas `font` shorthand string (`italic bold 30px Unna, Georgia, serif`) from the current `config.bold`/`config.italic`, used both to draw text and to `document.fonts.load()` the right variant. |
| `loadCaptionFonts()` | Awaits `document.fonts.load()` for both caption/date sizes before any caption drawing — Unna must be loaded before `ctx.font` can use it, or canvas silently falls back to the default serif font. |
| `runSession()` | First unlocks `shutterAudio` synchronously (play-then-pause, no `await` before it) — iOS Safari specifically only allows a media element's first `play()` to succeed when called directly inside the click handler, not later inside an async chain. Then orchestrates one full run: resets `shots`, loops 4× doing countdown → flash → capture, then `await`s `buildStrip()` and reveals the result screen. |
| Save/Share/Print handlers | `saveBtn` → `canvas.toDataURL('image/png')` download link. `shareBtn` → `navigator.share` with a `File` (only shown if `navigator.canShare` exists). `printBtn` → `window.print()`. `againBtn` → hides the result screen to shoot again. |
| `applySiteTheme(siteTheme)` / `applyConfig(cfg)` | Sets/removes `data-theme` on `<html>` (page chrome only) and syncs the customize form fields to a `config` object. |
| `encodeConfig(cfg)` / `decodeConfig(str)` / `readConfigFromURL()` | Pack/unpack `{siteTheme, caption, bold, italic, showDate}` to/from a base64-JSON `#c=` URL hash. `decodeConfig` validates `siteTheme` against the known list and clamps `caption` length — untrusted input, since it comes from a URL someone else generated. |
| `onConfigFieldChange()` | Fired on customize-field input/change; updates `config`, re-applies the site theme, updates `#dateStyleField` visibility, and re-runs `buildStrip()` live if a strip was already captured. |
| `renderQRCode(container, text)` | Renders a QR code via the vendored `QRCode` global, retrying increasing QR "type" (matrix size) until the payload fits — the library doesn't auto-size and throws on overflow otherwise. |
| `formatDate(date)` | Formats a `Date` per the current `dateStyle` (`'long'` → "Sep 14, 2026", `'numeric'` → "9.14.26"). Not called at all when `dateStyle === 'off'` — `computeCaptionLayout()` short-circuits to an empty `dateLine` in that case. |

**State** is minimal and intentionally not framework-managed: `stream` (MediaStream), `facingMode` (`'user' | 'environment'`, which camera is active — not part of `config`/the shared URL, purely a local device choice), `shots` (array of captured `<canvas>` elements), `config` (`{siteTheme, caption, bold, italic, showDate}` — the shared/lockable template), and `dateStyle` (`'long' | 'numeric'`, **not** part of `config`/the shared URL — see below), all module-level `let` variables closed over by the functions above.

**Constants** controlling capture geometry: `SHOT_COUNT = 4`, `FRAME_W/FRAME_H = 500×375` (4:3), `PADDING = 30` (outer margin around the photos — the "cut to size" white border), `GAP = 14` (between photos). Caption layout: `CAPTION_FONT_SIZE = 30`, `DATE_FONT_SIZE = 20`, `CAPTION_LINE_GAP = 10`, `CAPTION_PAD = 18` (the caption band height is computed from these based on which of caption/date are actually present, not a fixed constant).

### Custom template sharing

Feature: a user can set a site theme, caption (with bold/italic styling), and date visibility, then get a link/QR code that reproduces that exact template for whoever opens it — no login, no per-user frame upload.

Design choice (discussed with the project owner before building): **entirely static, config packed into the URL** rather than adding any backend/storage. Concretely: `{siteTheme, caption, bold, italic, showDate}` → `JSON.stringify` → UTF-8-safe base64 → `#c=<blob>` URL hash → QR code generated client-side from that URL via the vendored `qrcode.min.js`. Opening the link re-decodes the hash and calls `applyConfig()` before the user does anything else.

This was chosen over two heavier alternatives (still on the table if requirements grow):
- **Full accounts** — login + per-user frame management. Rejected as overkill for "share one template with someone."
- **Minimal serverless storage** — real image upload, stored server-side under a short ID. Would be needed if arbitrary custom frame *graphics* (not just theme/caption) become a requirement, since an uploaded image can't reasonably live inside a URL/QR code. This is the natural next step if that's ever wanted — see the conversation history around 2026-09-14 for the fuller tradeoff writeup.

Because of this choice, current "frames" are limited to the built-in `SITE_THEMES` list (`classic`, `noir`, `blush`, `mint`) plus free-text caption/style and a date toggle — not arbitrary uploaded graphics. Keep the encoded payload small (short strings/booleans only) so the QR code stays scannable.

`qrcode.min.js` is vendored (copied into the repo) rather than loaded from a CDN `<script src>` specifically to preserve the app's "everything runs on your device, no network dependency beyond the camera" property — a CDN script would make template rendering depend on a third party being up.

**Recipient lockdown.** When the page is opened via a shared link (i.e. `readConfigFromURL()` returns a config), `#customize` is hidden entirely (`customizeDetails.style.display = 'none'`) — the recipient gets the creator's theme/caption/style/date-visibility exactly as set, with no UI path to change any of it, including the caption. The one exception is **date format** (`'long'` / `'numeric'` / `'off'`, via `#dateStyleField`/`dateStyle`): this is deliberately kept *outside* `config`/the shared URL, so it's a free choice for whoever is using the page — creator or recipient — available once a strip has been captured, including the ability to hide the date entirely (`'off'`) even if the creator's locked `config.showDate` was `true`. `computeCaptionLayout()` treats the date as showing only when `config.showDate && dateStyle !== 'off'`. When there's no date line, the caption is drawn 2px lower (`drawCaptionLines()`'s `captionYOffset`) so it sits balanced in the caption band on its own. If a future request asks to lock the date format too, or to let recipients edit more than that, the config shape and the `customizeDetails.style.display = 'none'` line above are the places to revisit.

**Background music.** `#musicToggle` and `#bgMusic` (a looping `<audio>` pointed at `sounds/lofi-loop.mp3`) are shown on every page load, main page included — not gated on `shared`. Music **defaults to on**.

State is tracked as `musicUserDisabled` (a `let`, default `false` = "on") — an *intent* flag, not derived from `bgMusic.paused`. `updateMusicToggleLabel()` renders the toggle from this intent, so it correctly shows "on" immediately on load even before actual playback is technically possible (browsers block autoplay-with-sound until the visitor interacts with the page at all — expected, not a bug). `applyMusicIntent()` is the single place that both calls `play()`/`pause()` and refreshes the label; the toggle's click handler just flips `musicUserDisabled` and calls it.

Since default is "on" but autoplay is blocked pre-interaction, `tryStartMusicOnFirstInteraction()` listens for the visitor's first `pointerdown`/`keydown` **anywhere on the page** (not just the toggle) and starts playback then, unless `musicUserDisabled` is already `true`. It explicitly ignores clicks on `#musicToggle` itself and does not remove its listeners on such a click — the toggle's own handler already manages that click via `applyMusicIntent()`, and `pointerdown` fires before `click`, so without this exclusion the generic handler would start playback a beat before the toggle's own handler runs and (seeing `paused` already `false`) immediately reverses it. This was a real bug caught in testing, not a hypothetical — worth remembering if this logic is touched again.

Like `shutterAudio`, `bgMusic` is a plain `HTMLAudioElement`, not tied to the Web Audio API.

### Mobile-specific details already handled
- `playsinline muted autoplay` on `<video>` — required for iOS Safari to render the camera feed inline instead of forcing fullscreen.
- `viewport` meta tag for correct scaling.
- `facingMode: 'user'` requests the front camera by default, with `#flipCameraBtn` to switch to `'environment'` (back camera) — see `startCamera()`/`initCamera()` above. The button only appears when `enumerateDevices()` reports more than one camera, so it stays hidden on typical single-webcam laptops.

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
