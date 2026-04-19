# Display Debounce

## Problem

Transcription software (e.g., macOS Dictation) sometimes deletes a chunk of recently written text and rewrites it within ~500ms. This causes visible flicker in the caption display, making the reading experience unpleasant.

## Solution

Debounce the caption display render so it only updates after the textarea input has been stable for 500ms. Data handling (localStorage append, buffer trimming) remains immediate — only the visual render is delayed.

## Changes to `updateCaption()` in `index.html`

Currently `updateCaption()` does both data handling and display rendering on every `input` event. Split into two functions:

### `handleInput()` — runs immediately on every `input` event
- Append new text to localStorage (same logic as current)
- Trim textarea buffer if over limit (same logic as current)
- Update `previousLength` (same logic as current)
- Set/reset a 500ms debounce timer that calls `renderDisplay()`

### `renderDisplay()` — runs after 500ms of input stability
- Set `captionText.textContent` to current textarea value
- Scroll `captionDisplay` to bottom

## Constants

- `RENDER_DELAY = 500` — debounce delay in milliseconds, hardcoded

## What doesn't change

- localStorage behavior (append on every input, no delay)
- Buffer trimming (immediate, no delay)
- Export (pulls from localStorage, unaffected)
- Clear (resets everything immediately, also clears any pending render timer)
- Font size controls (unaffected)
- Fullscreen behavior (unaffected)
- Restore from localStorage on page load (renders immediately, no debounce)
