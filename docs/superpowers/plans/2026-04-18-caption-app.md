# Caption App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an offline-first PWA that displays dictated text as high-contrast, large-format rolling captions.

**Architecture:** Single-page app with three files: `index.html` (all HTML/CSS/JS inline), `manifest.json` (PWA metadata), and `service-worker.js` (offline caching). Text flows from a `<textarea>` into a rolling caption display. localStorage holds the full transcript; the textarea is trimmed for performance.

**Tech Stack:** Vanilla HTML/CSS/JS, Service Worker API, Fullscreen API, localStorage, Blob API for export.

---

## File Structure

```
caption/
├── index.html          # Entire app: HTML structure, inline CSS, inline JS
├── manifest.json       # PWA manifest (name, icons, theme, display mode)
├── service-worker.js   # Cache-first offline strategy
├── icon-192.png        # PWA icon 192x192
└── icon-512.png        # PWA icon 512x512
```

- `index.html` — single self-contained file. Contains three logical sections: `<style>` (all CSS), HTML body (caption display, toolbar, textarea), `<script>` (all JS). This is the only file with application logic.
- `manifest.json` — static PWA metadata. No logic.
- `service-worker.js` — caches the app shell on install, serves cache-first on fetch, cleans old caches on activate.
- `icon-192.png`, `icon-512.png` — PWA icons generated as simple "CC" text on black background.

---

### Task 1: PWA Icons

**Files:**
- Create: `icon-192.png`
- Create: `icon-512.png`

- [ ] **Step 1: Generate 192x192 icon**

Use a canvas-based Node script or an inline approach. Since we want no dependencies, create a simple HTML file to generate the icons, or use `sips`/ImageMagick if available. The simplest approach: create SVG icons and convert them.

Create a temporary script to generate the icons:

```bash
# Generate a 192x192 PNG with "CC" text on black background
python3 -c "
import struct, zlib

def create_png(width, height, filename):
    # Simple PNG with black background and white 'CC' text
    # We'll create a minimal black PNG - the text will be added via canvas in a helper HTML
    raw = []
    for y in range(height):
        row = b'\x00'  # filter byte
        for x in range(width):
            # Black pixel
            row += b'\x00\x00\x00\xff'
        raw.append(row)
    raw_data = b''.join(raw)

    def chunk(chunk_type, data):
        c = chunk_type + data
        crc = struct.pack('>I', zlib.crc32(c) & 0xffffffff)
        return struct.pack('>I', len(data)) + c + crc

    sig = b'\x89PNG\r\n\x1a\n'
    ihdr = struct.pack('>IIBBBBB', width, height, 8, 6, 0, 0, 0)
    compressed = zlib.compress(raw_data)

    with open(filename, 'wb') as f:
        f.write(sig)
        f.write(chunk(b'IHDR', ihdr))
        f.write(chunk(b'IDAT', compressed))
        f.write(chunk(b'IEND', b''))

create_png(192, 192, 'icon-192.png')
create_png(512, 512, 'icon-512.png')
print('Icons created')
"
```

This creates placeholder black PNGs. For proper "CC" icons, we'll use an HTML canvas helper:

```html
<!-- save as generate-icons.html, open in browser, right-click save images -->
<!-- DELETE this file after generating icons -->
<!DOCTYPE html>
<html>
<body>
<canvas id="c192" width="192" height="192"></canvas>
<canvas id="c512" width="512" height="512"></canvas>
<script>
function drawIcon(canvas, size) {
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = '#000000';
  ctx.fillRect(0, 0, size, size);
  ctx.fillStyle = '#FFFFFF';
  ctx.font = `bold ${size * 0.4}px -apple-system, Arial, sans-serif`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText('CC', size / 2, size / 2);
}
drawIcon(document.getElementById('c192'), 192);
drawIcon(document.getElementById('c512'), 512);

// Auto-download
['c192', 'c512'].forEach(id => {
  const canvas = document.getElementById(id);
  const size = id === 'c192' ? 192 : 512;
  const link = document.createElement('a');
  link.download = `icon-${size}.png`;
  link.href = canvas.toDataURL('image/png');
  link.click();
});
</script>
</body>
</html>
```

- [ ] **Step 2: Verify icons exist**

Run: `ls -la icon-192.png icon-512.png`
Expected: Both files exist with non-zero size.

- [ ] **Step 3: Commit**

```bash
git add icon-192.png icon-512.png
git commit -m "feat: add PWA icons"
```

---

### Task 2: PWA Manifest

**Files:**
- Create: `manifest.json`

- [ ] **Step 1: Create manifest.json**

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

- [ ] **Step 2: Verify JSON is valid**

Run: `python3 -c "import json; json.load(open('manifest.json')); print('valid')"`
Expected: `valid`

- [ ] **Step 3: Commit**

```bash
git add manifest.json
git commit -m "feat: add PWA manifest"
```

---

### Task 3: Service Worker

**Files:**
- Create: `service-worker.js`

- [ ] **Step 1: Create service-worker.js**

```javascript
const CACHE_NAME = 'caption-v1';
const ASSETS = [
  '/index.html',
  '/manifest.json',
  '/icon-192.png',
  '/icon-512.png'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(ASSETS))
  );
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((key) => key !== CACHE_NAME)
          .map((key) => caches.delete(key))
      )
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => cached || fetch(event.request))
  );
});
```

- [ ] **Step 2: Verify syntax**

Run: `node -c service-worker.js`
Expected: No output (no syntax errors).

- [ ] **Step 3: Commit**

```bash
git add service-worker.js
git commit -m "feat: add service worker for offline caching"
```

---

### Task 4: HTML Structure & CSS

**Files:**
- Create: `index.html`

This task creates the HTML structure and all CSS. No JavaScript yet — the page will be static but visually complete.

- [ ] **Step 1: Create index.html with HTML and CSS**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="theme-color" content="#000000">
  <title>Caption</title>
  <link rel="manifest" href="manifest.json">
  <link rel="icon" type="image/png" sizes="192x192" href="icon-192.png">
  <link rel="apple-touch-icon" href="icon-192.png">
  <style>
    *, *::before, *::after {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    html, body {
      height: 100%;
      background: #000000;
      color: #FFFFFF;
      font-family: -apple-system, BlinkMacSystemFont, Arial, sans-serif;
      overflow: hidden;
    }

    .app {
      display: flex;
      flex-direction: column;
      height: 100vh;
    }

    /* Caption display */
    .caption-display {
      flex: 1;
      display: flex;
      flex-direction: column;
      justify-content: flex-end;
      padding: 24px 32px;
      overflow: hidden;
      position: relative;
    }

    .caption-display::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 40%;
      background: linear-gradient(to bottom, #000000 0%, transparent 100%);
      pointer-events: none;
      z-index: 1;
    }

    .caption-text {
      font-size: 5vw;
      font-weight: 700;
      line-height: 1.5;
      word-wrap: break-word;
      overflow-wrap: break-word;
      white-space: pre-wrap;
    }

    /* Toolbar */
    .toolbar {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 8px 16px;
      background: #111111;
      border-top: 1px solid #333333;
      border-bottom: 1px solid #333333;
    }

    .toolbar-spacer {
      flex: 1;
    }

    .toolbar button {
      background: #333333;
      color: #FFFFFF;
      border: none;
      padding: 6px 14px;
      border-radius: 4px;
      font-size: 14px;
      cursor: pointer;
      font-family: inherit;
    }

    .toolbar button:hover {
      background: #444444;
    }

    .toolbar .btn-fullscreen {
      background: #2563eb;
    }

    .toolbar .btn-fullscreen:hover {
      background: #1d4ed8;
    }

    /* Text input area */
    .input-area {
      padding: 12px 16px;
      background: #111111;
    }

    .input-area textarea {
      width: 100%;
      min-height: 80px;
      background: #1a1a1a;
      color: #FFFFFF;
      border: 1px solid #333333;
      border-radius: 6px;
      padding: 12px;
      font-size: 16px;
      font-family: inherit;
      resize: vertical;
      outline: none;
    }

    .input-area textarea:focus {
      border-color: #2563eb;
    }

    /* Fullscreen mode */
    .app.fullscreen .toolbar,
    .app.fullscreen .input-area {
      display: none;
    }

    .app.fullscreen .caption-display {
      padding: 32px 48px;
    }

    .app.fullscreen .caption-text {
      font-size: 6vw;
    }

    /* Fullscreen hint */
    .fullscreen-hint {
      position: fixed;
      bottom: 16px;
      left: 50%;
      transform: translateX(-50%);
      color: #333333;
      font-size: 13px;
      opacity: 0;
      transition: opacity 0.5s;
      pointer-events: none;
      z-index: 10;
    }

    .fullscreen-hint.visible {
      opacity: 1;
    }

    /* Hidden textarea for fullscreen dictation */
    .hidden-input {
      position: absolute;
      opacity: 0;
      pointer-events: none;
      width: 0;
      height: 0;
    }
  </style>
</head>
<body>
  <div class="app" id="app">
    <div class="caption-display" id="captionDisplay">
      <div class="caption-text" id="captionText"></div>
    </div>

    <div class="toolbar" id="toolbar">
      <button class="btn-clear" id="btnClear">Clear</button>
      <button class="btn-export" id="btnExport">Export</button>
      <div class="toolbar-spacer"></div>
      <button class="btn-fullscreen" id="btnFullscreen">Fullscreen</button>
    </div>

    <div class="input-area" id="inputArea">
      <textarea
        id="captionInput"
        placeholder="Start typing or dictating..."
        autofocus
      ></textarea>
    </div>
  </div>

  <div class="fullscreen-hint" id="fullscreenHint">ESC to exit fullscreen</div>

  <script>
    // JS will be added in subsequent tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: Open in browser and verify layout**

Run: `open index.html`
Expected: Black page with empty caption area at top, toolbar with three buttons in the middle, textarea at the bottom. The textarea should have placeholder text "Start typing or dictating...".

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add HTML structure and CSS for caption app"
```

---

### Task 5: Caption Display Logic

**Files:**
- Modify: `index.html` (the `<script>` block)

This task wires the textarea to the caption display so typed/dictated text appears as rolling captions.

- [ ] **Step 1: Add caption rendering logic**

Replace the `<script>` block in `index.html` with:

```javascript
(function () {
  const captionInput = document.getElementById('captionInput');
  const captionText = document.getElementById('captionText');

  function updateCaption() {
    const text = captionInput.value;
    captionText.textContent = text;

    // Auto-scroll caption display to bottom
    const display = document.getElementById('captionDisplay');
    display.scrollTop = display.scrollHeight;
  }

  captionInput.addEventListener('input', updateCaption);

  // Keep textarea focused — click anywhere to refocus
  document.addEventListener('click', () => {
    captionInput.focus();
  });

  // Initial focus
  captionInput.focus();
})();
```

- [ ] **Step 2: Test in browser**

Run: `open index.html`
Test: Type text into the textarea. Verify it appears in the caption display area above with large white bold text. Verify clicking outside the textarea refocuses it.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: wire textarea input to caption display"
```

---

### Task 6: Buffer Trimming & localStorage Transcript

**Files:**
- Modify: `index.html` (the `<script>` block)

This task adds the rolling buffer (trim textarea to ~5,000 chars) and the full transcript in localStorage.

- [ ] **Step 1: Add buffer management and localStorage**

Replace the `<script>` block in `index.html` with the full updated version:

```javascript
(function () {
  const BUFFER_LIMIT = 5000;
  const STORAGE_KEY = 'caption-transcript';

  const captionInput = document.getElementById('captionInput');
  const captionText = document.getElementById('captionText');
  const captionDisplay = document.getElementById('captionDisplay');

  let previousLength = 0;

  function updateCaption() {
    const currentValue = captionInput.value;

    // Append only the new text to localStorage
    if (currentValue.length > previousLength) {
      const newText = currentValue.slice(previousLength);
      const existing = localStorage.getItem(STORAGE_KEY) || '';
      localStorage.setItem(STORAGE_KEY, existing + newText);
    }

    // Trim textarea if it exceeds the buffer limit
    if (currentValue.length > BUFFER_LIMIT) {
      const trimmed = currentValue.slice(currentValue.length - BUFFER_LIMIT);
      // Find the first space to trim at a word boundary
      const firstSpace = trimmed.indexOf(' ');
      const cleanTrimmed = firstSpace > -1 ? trimmed.slice(firstSpace + 1) : trimmed;
      captionInput.value = cleanTrimmed;
    }

    previousLength = captionInput.value.length;

    // Update caption display
    captionText.textContent = captionInput.value;
    captionDisplay.scrollTop = captionDisplay.scrollHeight;
  }

  captionInput.addEventListener('input', updateCaption);

  // Restore from localStorage on load (use the tail of the transcript)
  function restoreFromStorage() {
    const transcript = localStorage.getItem(STORAGE_KEY);
    if (transcript) {
      const tail = transcript.length > BUFFER_LIMIT
        ? transcript.slice(transcript.length - BUFFER_LIMIT)
        : transcript;
      captionInput.value = tail;
      previousLength = tail.length;
      captionText.textContent = tail;
      captionDisplay.scrollTop = captionDisplay.scrollHeight;
    }
  }

  restoreFromStorage();

  // Keep textarea focused
  document.addEventListener('click', () => {
    captionInput.focus();
  });

  captionInput.focus();
})();
```

- [ ] **Step 2: Test in browser**

Run: `open index.html`
Tests:
1. Type text — verify it appears in caption display.
2. Refresh the page — verify the text is restored from localStorage.
3. Open browser dev tools → Application → Local Storage — verify `caption-transcript` key exists with the typed text.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add buffer trimming and localStorage transcript"
```

---

### Task 7: Toolbar Actions (Clear, Export, Fullscreen)

**Files:**
- Modify: `index.html` (the `<script>` block)

This task wires up all three toolbar buttons.

- [ ] **Step 1: Add toolbar button handlers**

Add the following code inside the IIFE, after the `captionInput.focus()` line at the bottom of the existing script:

```javascript
  // --- Clear ---
  const btnClear = document.getElementById('btnClear');
  btnClear.addEventListener('click', (e) => {
    e.stopPropagation(); // Prevent refocus-click from interfering
    captionInput.value = '';
    captionText.textContent = '';
    localStorage.removeItem(STORAGE_KEY);
    previousLength = 0;
    captionInput.focus();
  });

  // --- Export ---
  const btnExport = document.getElementById('btnExport');
  btnExport.addEventListener('click', (e) => {
    e.stopPropagation();
    const transcript = localStorage.getItem(STORAGE_KEY) || '';
    if (!transcript) return;

    const now = new Date();
    const pad = (n) => String(n).padStart(2, '0');
    const timestamp = [
      now.getFullYear(),
      pad(now.getMonth() + 1),
      pad(now.getDate()),
      '-',
      pad(now.getHours()),
      pad(now.getMinutes()),
      pad(now.getSeconds())
    ].join('');
    const filename = 'caption-transcript-' + timestamp + '.txt';

    const blob = new Blob([transcript], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
    captionInput.focus();
  });

  // --- Fullscreen ---
  const app = document.getElementById('app');
  const btnFullscreen = document.getElementById('btnFullscreen');
  const fullscreenHint = document.getElementById('fullscreenHint');

  btnFullscreen.addEventListener('click', (e) => {
    e.stopPropagation();
    document.documentElement.requestFullscreen();
  });

  document.addEventListener('fullscreenchange', () => {
    if (document.fullscreenElement) {
      app.classList.add('fullscreen');
      // Show hint briefly
      fullscreenHint.classList.add('visible');
      setTimeout(() => {
        fullscreenHint.classList.remove('visible');
      }, 3000);
    } else {
      app.classList.remove('fullscreen');
      fullscreenHint.classList.remove('visible');
    }
    captionInput.focus();
  });
```

- [ ] **Step 2: Test Clear button**

Run: `open index.html`
Test: Type text, click Clear. Verify textarea is empty, caption display is empty, and localStorage `caption-transcript` is removed (check dev tools).

- [ ] **Step 3: Test Export button**

Test: Type some text, click Export. Verify a `.txt` file downloads with the correct content and a timestamped filename like `caption-transcript-20260418-143022.txt`.

- [ ] **Step 4: Test Fullscreen button**

Test: Click Fullscreen. Verify:
1. Page enters fullscreen mode.
2. Toolbar and textarea are hidden.
3. Caption text is larger (`6vw`).
4. "ESC to exit fullscreen" hint appears briefly then fades.
5. Pressing Escape exits fullscreen and restores normal layout.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add clear, export, and fullscreen toolbar actions"
```

---

### Task 8: Fullscreen Dictation Focus

**Files:**
- Modify: `index.html` (the `<script>` block)

In fullscreen mode, the textarea is hidden via CSS but must retain focus so dictation software can still feed text into it. This task ensures focus is maintained robustly.

- [ ] **Step 1: Add focus management for fullscreen**

Add the following inside the IIFE, after the fullscreen change handler:

```javascript
  // In fullscreen, aggressively maintain focus on the textarea
  // Dictation software needs a focused input element to work
  document.addEventListener('keydown', () => {
    if (document.activeElement !== captionInput) {
      captionInput.focus();
    }
  });

  // Re-focus on visibility change (e.g., switching tabs and back)
  document.addEventListener('visibilitychange', () => {
    if (!document.hidden) {
      captionInput.focus();
    }
  });
```

- [ ] **Step 2: Test focus in fullscreen**

Test: Enter fullscreen, click around the page. Verify:
1. The textarea (though hidden) remains the active element — check via dev tools `document.activeElement`.
2. Typing still produces caption text in fullscreen.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: maintain textarea focus in fullscreen for dictation"
```

---

### Task 9: Service Worker Registration

**Files:**
- Modify: `index.html` (the `<script>` block)

- [ ] **Step 1: Add service worker registration**

Add the following at the very end of the `<script>` block, outside the IIFE:

```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('service-worker.js');
}
```

- [ ] **Step 2: Test PWA registration**

Serve the app locally (service workers require HTTP, not `file://`):

Run: `python3 -m http.server 8000 --directory /Users/brian.buchalter/workspace/caption`

Open `http://localhost:8000` in Chrome. Open dev tools → Application → Service Workers. Verify:
1. Service worker is registered and active.
2. Application → Cache Storage shows `caption-v1` with the cached assets.
3. Check "Offline" in Network tab, refresh — app still works.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: register service worker for offline support"
```

---

### Task 10: Final Integration Test

**Files:** None (testing only)

- [ ] **Step 1: Serve the app**

Run: `python3 -m http.server 8000 --directory /Users/brian.buchalter/workspace/caption`

- [ ] **Step 2: Full workflow test**

Open `http://localhost:8000` in browser. Run through this checklist:

1. **Caption display:** Type text → appears as large white text on black background.
2. **Rolling text:** Type enough text to fill the display → older lines scroll up and fade under the gradient.
3. **Buffer trimming:** Paste a very long string (>5000 chars) → textarea trims without breaking the display.
4. **localStorage:** Refresh the page → text is restored.
5. **Clear:** Click Clear → textarea, display, and localStorage are all empty.
6. **Export:** Type text, click Export → `.txt` file downloads with correct content.
7. **Fullscreen:** Click Fullscreen → toolbar and input hide, captions fill screen, hint appears and fades, ESC exits.
8. **Dictation in fullscreen:** Enter fullscreen, keep typing → text still appears (simulates dictation).
9. **Offline:** In dev tools, check "Offline" in Network tab, refresh → app works.
10. **PWA install:** Check for install prompt in browser address bar.

- [ ] **Step 3: Commit any fixes if needed**

If any issues were found and fixed:
```bash
git add -A
git commit -m "fix: address issues found in integration testing"
```
