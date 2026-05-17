# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install      # Install dependencies
npm start        # Run the Electron app in development
npm run make     # Build distributable installers
npx prettier --write .   # Format code before committing (4-space indent, width 150, semicolons, single quotes)
```

No linting or automated test suite exists yet (`npm run lint` is a no-op, `npm test` does not exist).

## Architecture

This is an **Electron app** using **Lit Web Components** (not React — migration is planned but not done). Entry points:

- **`src/index.js`** — Electron main process. Sets up IPC handlers for storage, general commands, and Gemini AI. Creates the single `BrowserWindow`.
- **`src/index.html`** — Renderer entry. Loads Lit, marked.js, highlight.js, then `CheatingDaddyApp.js` as an ES module. The custom element `<sales-master-app>` mounts here.
- **`src/utils/renderer.js`** — Renderer-side logic. Defines the global `window.cheatingDaddy` object that all Lit components call into. Handles screen/audio capture, platform detection (`isMacOS`, `isLinux`), theme system, and the `storage` IPC wrapper.
- **`src/utils/window.js`** — Window creation, global keyboard shortcuts, and IPC handlers for window operations (minimize, toggle visibility, click-through).
- **`src/utils/gemini.js`** — All AI provider logic: Google Gemini Live, Groq, Ollama, and cloud mode. Session management and audio/image streaming.
- **`src/storage.js`** — Main-process file storage (JSON files). Config lives at `sales-master-config/` in the OS app data dir.

### UI Component Tree

```
<sales-master-app>  (CheatingDaddyApp.js — app shell, routing, Windows titlebar)
  ├── Windows titlebar (fixed, 32px, drag region + min/max/close buttons)
  ├── Sidebar (nav items, hidden in live/assistant mode)
  └── Content area
        ├── <main-view>        — API key entry, profile select, start session
        ├── <assistant-view>   — Live AI response display
        ├── <customize-view>   — Settings, keybinds, theme
        ├── <ai-customize-view>— Custom system prompt per profile
        ├── <history-view>     — Past sessions
        ├── <help-view>        — Keyboard shortcuts reference
        ├── <feedback-view>    — Feedback form
        └── <onboarding-view>  — First-run setup (fullscreen)
```

### IPC Pattern

Components never call `ipcRenderer` directly — they call `window.cheatingDaddy.*` which is the renderer-side API defined in `renderer.js`. That object wraps all IPC calls. Main-process handlers are registered in `src/index.js` and `src/utils/window.js`.

### Platform-Specific Audio

- **macOS**: `SystemAudioDump` binary (bundled in `src/assets/`) captures system audio via IPC; screen via `getDisplayMedia`
- **Windows**: `getDisplayMedia` with loopback audio
- **Linux**: `getDisplayMedia` for screen; microphone only for audio

Audio is processed at 24 kHz, 16-bit PCM, sent to Gemini Live in 100ms chunks.

### AI Providers

Three modes selectable in the UI (stored in preferences as `providerMode`):
- **`byok`** — User's own Gemini API key (default). Optionally also a Groq key.
- **`local`** — Ollama (local LLM) + Whisper (local STT via `@huggingface/transformers`)
- **`cloud`** — Backend cloud service (UI hidden, wiring present in code)

### Storage Layout

All data stored as JSON files under `sales-master-config/` (in `AppData/Roaming` on Windows, `~/Library/Application Support` on macOS, `~/.config` on Linux):
- `config.json` — onboarded flag, layout mode
- `credentials.json` — API keys
- `preferences.json` — profile, language, audio mode, theme, keybinds etc.
- `keybinds.json` — custom keybind overrides
- `history/` — one JSON file per session

## Key Conventions

- **Prettier enforced**: run `npx prettier --write .` before every commit. Config is in `.prettierrc`.
- **No TypeScript yet**: all existing code is plain JS. New files should be JS until the React/TS migration begins.
- **`window.cheatingDaddy`** is the global API — components should call this, not `ipcRenderer` directly.
- **Commit messages**: use `feat:` / `fix:` / `chore:` prefixes. Keep them short.
- The app is **frameless** (`frame: false`) with a custom Windows-style titlebar rendered in the renderer (32px, `Sales Master` label left, min/max/close right).
- When merging upstream changes from `sohzm/cheating-daddy`, run the app locally to verify it still builds before pushing.
