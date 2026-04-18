# Caption — Offline-First Live Caption Display

## Overview

A Progressive Web App for live manual captioning. Dictation software feeds text into a hidden input, and the app displays it as high-contrast, large-format rolling captions — like closed captioning on a screen or projector. The app works entirely offline after first visit.

## Use Case

A presenter or speaker uses dictation software (e.g., macOS Dictation, Dragon) to transcribe speech in real time. The app displays the transcribed text as large, readable captions on a screen visible to the audience. The typist/dictator and audience share the same device (single-device use only).

## App Modes

### Normal Mode (Demo/Setup)

- **Top area (~70% of viewport):** Caption display showing rolling text, newest lines at the bottom, older lines fading out as they scroll up.
- **Toolbar:** A horizontal bar between the caption display and text input with: Fullscreen button, Clear button, Export button.
- **Bottom area (~30% of viewport):** A `<textarea>` where dictation software feeds text. Visible and editable for setup and demo purposes.

### Fullscreen Mode (Presentation)

- Caption display fills the entire viewport.
- The `<textarea>` remains in the DOM and retains focus so dictation continues to work, but it is visually hidden (`position: absolute; opacity: 0; pointer-events: none`).
- Toolbar is hidden.
- A subtle hint ("ESC to exit fullscreen") is shown briefly on entering fullscreen, then fades out.
- Exit via `Escape` key (browser Fullscreen API default).

### Entering/Exiting Fullscreen

- Fullscreen button in toolbar triggers `document.documentElement.requestFullscreen()`.
- `Escape` exits fullscreen (handled by browser).
- The app listens to `fullscreenchange` events to toggle between modes.

## Visual Style

Single visual style, no presets or customization:

| Property        | Value                                          |
|-----------------|-------------------------------------------------|
| Background      | `#000000` (pure black)                          |
| Text color      | `#FFFFFF` (pure white)                          |
| Font family     | `-apple-system, BlinkMacSystemFont, Arial, sans-serif` |
| Font weight     | `700` (bold)                                    |
| Font size       | `5vw` (scales with viewport width)              |
| Line height     | `1.5`                                           |
| Letter spacing  | Normal                                          |

## Caption Display Behavior

### Rolling Text

- The caption display area shows the most recent lines of text.
- Text wraps naturally at word boundaries based on viewport width (CSS `word-wrap: break-word`).
- Older lines scroll up and fade out using an opacity gradient (most recent line fully opaque, older lines progressively more transparent).
- The display auto-scrolls to keep the newest text visible at the bottom.

### Line Visibility

- The number of visible lines is determined dynamically by the viewport height and font size.
- No fixed line count — the display simply fills available space with the most recent text.

## Text Input & Buffer Management

### Input

- A `<textarea>` element receives text from dictation software.
- The textarea must maintain focus at all times (especially in fullscreen mode) so dictation input is not interrupted.
- Re-focus the textarea on click anywhere in the document.

### Rolling Buffer (Performance)

- The `<textarea>` value is trimmed to the last ~5,000 characters to keep the DOM performant during long sessions (1+ hours of continuous dictation).
- Trimming happens on each input event when the length exceeds the threshold.
- Trim at a word boundary to avoid cutting a word in half.

### Full Transcript (localStorage)

- On each input event, new text is appended to a localStorage key (`caption-transcript`).
- This stores the complete, untrimmed transcript of the entire session.
- The Export button pulls from localStorage, not from the textarea.
- The Clear button clears both the textarea and localStorage.
- localStorage survives accidental page refreshes.

## Toolbar Controls

Three buttons, visible only in normal mode:

### Fullscreen Button

- Label: "Fullscreen" (or a fullscreen icon)
- Action: Enters browser fullscreen mode via the Fullscreen API.

### Clear Button

- Label: "Clear" (or a trash/reset icon)
- Action: Clears the textarea, clears the caption display, clears the localStorage transcript.
- No confirmation dialog — it's a caption app, not a document editor.

### Export Button

- Label: "Export" (or a download icon)
- Action: Downloads the full transcript from localStorage as a plain `.txt` file.
- Filename: `caption-transcript-YYYY-MM-DD-HHMMSS.txt` (timestamped).
- Uses the `Blob` + `URL.createObjectURL` + `<a download>` pattern. No server needed.

## Keyboard Shortcuts

| Shortcut | Action              |
|----------|---------------------|
| `Escape` | Exit fullscreen     |

Fullscreen entry is via the toolbar button only (no keyboard shortcut to avoid conflicts with dictation software).

## File Structure

```
caption/
├── index.html          # Entire app: HTML, CSS, JS all inline
├── manifest.json       # PWA manifest
└── service-worker.js   # Caches files for offline use
```

### index.html

Single self-contained file with:
- Inline `<style>` block with all CSS
- HTML structure: caption display, toolbar, textarea
- Inline `<script>` block with all JS (input handling, buffer management, localStorage, fullscreen, export)
- PWA registration (`navigator.serviceWorker.register`)
- Link to `manifest.json`

### manifest.json

```json
{
  "name": "Caption",
  "short_name": "Caption",
  "description": "Offline-first live caption display",
  "start_url": "/index.html",
  "display": "standalone",
  "background_color": "#000000",
  "theme_color": "#000000",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

Icons will be simple generated SVG-based icons (a "CC" symbol or text icon) at required PWA sizes (192x192, 512x512).

### service-worker.js

- On `install`: caches `index.html`, `manifest.json`, and icon files.
- On `fetch`: serves from cache first, falls back to network (cache-first strategy).
- On `activate`: cleans up old caches if the cache version changes.

## Deployment

Any static file host works: GitHub Pages, Netlify, Vercel, Cloudflare Pages, or even opening `index.html` from disk (though service worker requires `localhost` or HTTPS for PWA features).

## Out of Scope (v1)

- Multi-device sync (typing on one device, displaying on another)
- Speech-to-text built into the app (relies on external dictation software)
- Multiple presets or visual customization
- User accounts or cloud storage
- Real-time collaboration
