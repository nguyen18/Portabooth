# Portabooth

A browser-based photobooth — four frames, one strip, all yours. No app to install, nothing uploaded anywhere; everything runs on your own device.

**Live: [portabooth.studio](https://portabooth.studio)**

## Features

- Countdown-timed 4-shot capture, with a camera flash and shutter sound
- Front/back camera switching on devices with more than one camera
- Customize before you shoot: site color theme, caption text (bold/italic), whether to show the date
- After shooting: pick a date style (or turn it off) and a photo filter (Black & White, Sepia, Saturated), all live-updating on the strip
- Generate a shareable link + QR code that hands someone your exact template — they open it, shoot their own strip, no account needed
- Background music with an on/off toggle
- Save straight to Photos on mobile, or to Files on desktop; share via the native share sheet; print two copies side by side on a 4"×6" sheet, ready to cut

## Running it locally

It's a single static site — no build step, no dependencies to install. Serve the folder with anything that speaks HTTP and open it:

```
python3 -m http.server 8934
```

then visit `http://localhost:8934/`. The camera requires a secure context, so `localhost` works but opening `index.html` directly as a `file://` URL will not.

## Under the hood

Plain HTML/CSS/JS — no framework, no build tooling. See [ARCHITECTURE.md](ARCHITECTURE.md) for a full deep dive into how it's put together, the design decisions behind it, and known issues.
