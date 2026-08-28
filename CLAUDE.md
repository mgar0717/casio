# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This is a static, no-build, no-dependency site containing two unrelated personal web apps, deployed as-is (e.g. via GitHub Pages — `manifest.json`'s `start_url` is `/casio/`). There is no `package.json`, build tool, test suite, or CI — every page is a single self-contained HTML file with inline `<style>` and `<script>` (no separate `.css`/`.js` files, no bundler, no framework).

1. **동암교회 제자훈련 암송 (Scripture memorization player)** — `index.html` (root) and `swgame/../player3.html` at the repo root
2. **선우의 학습 게임 (Kids' learning games)** — everything under `swgame/`

These two apps share no code; treat them as independent projects that happen to live in one repo.

## Development workflow

There is no build/lint/test tooling. To work on a page:

- Open the `.html` file directly in a browser, or serve the directory locally (e.g. `python3 -m http.server`) since the memorization player uses `<script type="module">` for Firebase (ES module imports require `http(s)://`, not `file://`).
- Edit HTML/CSS/JS in place inside the single file — there's nothing to compile.
- `index.html` and `player3.html` are large (multi-MB) because each embeds its narration audio track as a base64 `data:audio/mpeg;base64,...` URI directly in the page. Avoid opening/searching these files whole; the base64 blob dominates the file and isn't meant to be read/edited. Use `grep -n` / targeted line ranges to jump to the actual markup/script (the real content is a few hundred lines; the rest of the byte count is the embedded audio).

## App 1: Scripture memorization player (`index.html`, `player3.html`)

A single-page Korean-language audio player for memorizing catechism/scripture passages ("동암교회 제자훈련" — a discipleship-training curriculum), built for mobile (PWA via `manifest.json` + `icon-192.png`/`icon-512.png`).

- **`index.html`** = 1학기 (semester 1): 1권 1과 ~ 2권 14과
- **`player3.html`** = 2학기 (semester 2): 3권 1과 ~ 3권 12과
- The two files are near-duplicates of each other (same CSS/markup/logic), differing in: the embedded audio track, the `SECTIONS`/`TS` data (see below), the semester-toggle links between the two pages, and minor Firebase presence-tracking behavior (semester 1 registers presence via `set`+`onDisconnect`; semester 2 only reads presence, it doesn't register). **When fixing a bug or tweaking shared UI/logic, check whether the same change is needed in both files** — there's no shared include.

Structure inside each file:
- `<script type="module">` (top): Firebase Realtime Database setup (`firebaseConfig`, hardcoded API key — this is a client-side Firebase web config, expected to be public) used only for a live "who's listening now" presence counter (`presence/<sessionId>` node, self-removed via `onDisconnect`).
- `<script>` (main, non-module): all player logic, in this order:
  - `SECTIONS` — array of `{id, title, text}` per lesson/passage (the verse text and reference, Korean).
  - `TS` — parallel array of audio timestamps (seconds) marking where each section starts in the single embedded audio track. Section boundaries are timestamp-driven, not separate audio files.
  - Playback controls: play/pause, seek (`doSeek`), skip, speed (`setSp`), section loop / full loop / A-B loop (`togSecLoop`/`togAllLoop`/`togAbMode`).
  - Lyrics/subtitle rendering: builds one `.lyric-item` per `SECTIONS` entry, highlights the active one via `updateActiveSection()` (driven by `audio.timeupdate` + `TS`).
  - **Recitation checker** (speech-to-text grading): uses the Web Speech API (`webkitSpeechRecognition`/`SpeechRecognition`, `lang="ko-KR"`, Chrome-only — the code explicitly checks for and alerts about this). Flow: `togMic` → `startMic` → `launchRecog` (auto-restarts recognition since Chrome's recognizer stops after silence) → `finalizeResult` → `showResult`, which fuzzy-matches spoken text against the section's reference text (Levenshtein-based `wordSim`/`editDist`, plus Korean-specific normalization: `normNum` for digit-to-Korean-numeral conversion, `applyGeoeo` for common archaic/formal verb-ending misrecognitions, `applyPhonetic` for single-character phonetic confusions) and renders a color-coded diff + match percentage.
- To add/edit a lesson passage: edit the `SECTIONS` array entry (id/title/text) and add the matching start timestamp to `TS` (keep them in sync and in ascending order) — but note the timestamp only matters if the embedded audio track actually contains that passage at that offset, so audio edits and `TS` edits must be done together.
- `GEOEO`/`PHONETIC` maps are targeted fixes for specific Korean speech-recognition misreadings observed in practice — extend them the same way (misheard-form → correct-form) rather than generalizing the matching algorithm.

## App 2: Kids' learning games (`swgame/`)

A small hub (`swgame/index.html`) linking to three independent quiz-style games, all following the same pattern:

- `swgame/math_game.html` — arithmetic drills (operator/difficulty selection → timed multiple-choice or input questions)
- `swgame/english_game.html` — English vocabulary/word practice (category/direction/difficulty selection)
- `swgame/science_game.html` — science quiz (category/difficulty selection)

Each game is a self-contained single-page "screen" app with no routing library: sections are plain `<div class="screen">` elements toggled via `showScreen(name)`/`.active` class. Common per-game flow: `select*()` (mode/difficulty pickers) → `startGame()` → `loadQ()` (loads next question) → `startTimer()` (per-question countdown) → `checkAns()`/`checkInput()` → `handleResult()` → `nextQ()` or `endGame()` → `replayGame()`/`goHome()`. Question pools are generated or selected in-file (`genQ`/`makeQs`/`makeChoices`/`getPool`) — there's no external question bank or persistence (no `localStorage`); state resets on reload.

When adding a new game, follow this same self-contained single-file structure and the `screen`/`showScreen` pattern rather than introducing shared JS/CSS files or a build step, to stay consistent with the rest of the repo.
