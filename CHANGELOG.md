# Changelog

## v0.1.0 (2026-07-01)

### ✨ Features

- **Activity Detection**: Global keyboard & mouse monitoring via Windows low-level hooks (WH_KEYBOARD_LL + WH_MOUSE_LL) and polling thread (GetAsyncKeyState)
- **Focus State Machine**: 8-state FSM (idleNoTask → focusedActive → focusedQuiet → stallSuspected → stallLong → intervening → backoff → review)
- **Graduated Intervention**: L1 stall reminder → L2 dialog → L3 alarm audio, with backoff mechanism to prevent annoyance
- **Task Management**: 3-task tree structure with sub-modules, anchor focus to any sub-module
- **Session Timer**: Per-task focus timer with pause/resume
- **Custom Audio**: User-selectable alert sounds for L1/L3 (MP3/WAV)
- **Audio Alert**: Plays configurable alert sounds when stall detected
- **Floating Widget**: Always-on-top transparent window with persona avatar
- **Intervention Popup**: Separate popup window for L3 alerts
- **SQLite Storage**: Local database for episode tracking and preferences
- **Dual Mode**: Runs as Tauri desktop app or in browser for development

### 🛠 Tech Details

- Desktop shell: **Tauri 2** (Rust + WebView2)
- Frontend: **React 19 + TypeScript + Vite 6**
- Backend: **Rust** with windows-sys for input hooks
- State management: Pure function FSM with useReducer (no XState)
- Database: **SQLite** via rusqlite (bundled)
- Audio: HTML5 Audio element + base64 data URL for custom files
