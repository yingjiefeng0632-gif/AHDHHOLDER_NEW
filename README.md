# 🧘 ADHDHolder

> **Your desktop focus copilot — judgment-free, rhythm-aware, always-on-top.**
>
> **你的桌面专注副驾驶 — 零评判、感知节奏、常驻桌面。**

** contact e-mail yingjiefeng0632@gmail.com **
[![Rust](https://img.shields.io/badge/Rust-1.75%2B-orange)](https://www.rust-lang.org/)
[![Tauri](https://img.shields.io/badge/Tauri-2-blueviolet)](https://v2.tauri.app/)
[![React](https://img.shields.io/badge/React-19-blue)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.7-strict)](https://www.typescriptlang.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## 📖 What is ADHDHolder?

ADHDHolder is a tiny floating desktop app that helps you stay focused. It silently watches your keyboard & mouse rhythm (never the content). When you've been idle too long — lost in thought, distracted, or stuck — it gently pulls you back with a sound and a popup. That's it.

**Philosophy**: One alarm. No nagging. No judgment.

> ADHDHolder 是一款常驻桌面的微型浮窗应用。它安静感知你的键鼠**活动节奏**（不记录任何内容），当你走神太久没动，它用一声提示音 + 一个弹窗轻轻提醒你。没有多层分级，没有唠叨追问——就一声。

---

## ✨ Features

- 🎯 **Task Tree** — 3 tasks × sub-modules, click to anchor "I'm working on this"
- 🧠 **Activity Detection** — Windows global keyboard/mouse hooks (WH_KEYBOARD_LL + WH_MOUSE_LL), content-blind
- ⏰ **Single Alarm** — Set a timeout (default 3 min). No input → alarm sounds → popup appears → reply to stop
- 🔊 **Custom Audio** — Choose your own alarm sound (WAV/MP3/OGG/FLAC/AAC)
- ⏱ **Session Timer** — Automatic per-task focus timer with pause/resume
- 🔒 **Privacy-First** — Never records keystrokes, window titles, or screen content. All data stored locally (SQLite).
- 🪟 **Always-on-Top Widget** — Floating transparent window, 380×64 collapsed, hover to expand

---

## 🖥️ Screenshots

| Collapsed | Expanded | Alarm Popup |
|:---:|:---:|:---:|
| *380×64 floating widget* | *Task panel + sub-modules* | *Reminder popup* |

*(screenshots coming soon — PR welcome!)*

---

## 📦 Download & Install

### Option 1: Download from GitHub Releases (Recommended)

1. Go to the [**AHDHHOLDER_NEW**](https://github.com/yingjiefeng0632-gif/AHDHHOLDER_NEW) page
2. Download the latest installer for Windows:

   | File | Description |
   | `adhdholder.exe` | **Portable** — double-click to run, no installation needed |

3. A small floating widget appears on your desktop — you're ready to go!

### System Requirements

| Requirement | Details |
|---|---|
| **OS** | Windows 10 / Windows 11 (64-bit) |
| **WebView2 Runtime** | Pre-installed on Windows 11. Windows 10 will auto-install if missing. |
| **Other** | No dependencies — everything is bundled in the app |

---

## ⚙️ Configuration

Open **Settings** (gear icon ⚙ on the widget) to adjust:

| Setting | Default | Range | Description |
|---|---|---|---|
| **Idle Timeout** | 180s (3 min) | 10s – 1800s | How long without input before the alarm triggers |
| **Alarm Sound** | Default `.wav` | Custom file | Choose your own WAV/MP3/OGG/FLAC/AAC file |
| **Volume** | 100% | 0% – 100% | Alarm sound volume |

---

## 🚀 Build from Source

### Prerequisites

- [Node.js](https://nodejs.org/) ≥ 20
- [Rust](https://www.rust-lang.org/) ≥ 1.75
- Windows 10/11 with [WebView2](https://developer.microsoft.com/en-us/microsoft-edge/webview2/)

### Build

```bash
# 1. Clone the repo
git clone https://github.com/yingjiefeng0632-gif/AHDHHOLDER_NEW.git
cd AHDHHOLDER_NEW

# 2. Install frontend dependencies
npm install

# 3. Build the full desktop app (frontend + Rust backend + installer)
npx tauri build
```

Build outputs will be in `src-tauri/target/release/bundle/`:
- `msi/` — MSI installer
- `nsis/` — NSIS setup `.exe`

### Development

```bash
# Browser-only mode (hot reload, mock idle simulation)
npm run dev
# Opens: http://localhost:1420

# Full Tauri desktop mode (hot reload, real input hooks)
npx tauri dev
```

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────┐
│  Floating UI       React 19 + TS        │  ← User declares focus here
├─────────────────────────────────────────┤
│  State Machine     4-state FSM          │  ← useReducer (pure function)
├─────────────────────────────────────────┤
│  Perception        Input rhythm only     │  ← Windows global hooks
├─────────────────────────────────────────┤
│  Backend           Rust + SQLite        │  ← Tauri 2
└─────────────────────────────────────────┘
```

**FSM States**: `idleNoTask` → `focusedActive` → `alarming` → `review`

| State | Meaning |
|---|---|
| `idleNoTask` | No task anchored — widget is idle |
| `focusedActive` | User is working (input detected within threshold) |
| `alarming` | Idle timeout exceeded — alarm sounds, popup shows |
| `review` | Sub-module completed — show review summary |

### Input Monitoring (Dual-Thread)

```
Thread A: WH_KEYBOARD_LL + WH_MOUSE_LL hooks (event-driven)
Thread B: GetAsyncKeyState polling (backup, every 1s)
         → compute idle_seconds → emit("activity-update")
```

---

## 📁 Project Structure

```
ADHDHolder/
├── src/                          # Frontend (React + TypeScript)
│   ├── main.tsx                  # React entry
│   ├── App.tsx                   # Main app component
│   ├── types.ts                  # TypeScript types
│   ├── machines/
│   │   └── focusMachine.ts       # 4-state FSM
│   ├── hooks/
│   │   ├── useActivity.ts        # Input signal listener
│   │   ├── useFocusMachine.ts    # FSM integration
│   │   ├── useInterventionPopup.ts # Popup lifecycle
│   │   ├── useFloatingWindow.ts  # Window resize/state
│   │   └── useSessionTimer.ts    # Focus timer
│   ├── components/
│   │   ├── FloatingWidget.tsx    # Collapsed widget
│   │   ├── ExpandedPanel.tsx     # Expanded panel
│   │   ├── SettingsPanel.tsx     # Settings UI
│   │   ├── InterventionPopup.tsx # Alarm popup window
│   │   └── ...
│   └── styles/
│       └── index.css
│
├── src-tauri/                    # Backend (Rust)
│   ├── src/
│   │   ├── main.rs               # App entry + Tauri setup
│   │   ├── commands.rs           # IPC commands
│   │   ├── state.rs              # App state
│   │   ├── input_monitor.rs      # Windows input hooks
│   │   └── db.rs                 # SQLite layer
│   ├── Cargo.toml
│   └── tauri.conf.json
│
├── public/
│   └── testaudio.wav             # Default alarm sound
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── .gitignore
├── LICENSE                       # MIT
├── CHANGELOG.md
└── README.md
```

---

## 🔒 Privacy

ADHDHolder is designed with **privacy as a built-in feature**:

- **Content-blind**: Only tracks timestamps — never records keystrokes, window titles, or screen content
- **Local-first**: All data (episodes, preferences) stored in a local SQLite database
- **No telemetry**: No analytics, no phoning home — ever
- **No account required**: Works entirely offline

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Desktop Shell | **Tauri 2** (Rust + WebView2) |
| Frontend | **React 19** · **TypeScript 5.7** · **Vite 6** |
| Backend | **Rust** · **windows-sys** · **rusqlite** |
| State Machine | Pure function FSM (useReducer) |
| Database | **SQLite** (bundled) |
| Audio | HTML5 Audio + base64 data URL |

---

## 🤝 Contributing

Contributions welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Quick start:
```bash
git clone https://github.com/yingjiefeng0632-gif/AHDHHOLDER_NEW.git
cd AHDHHOLDER_NEW
npm install
npx tauri dev
```

---

## 📄 License

[MIT](LICENSE) © 2026 ADHDHolder Contributors

---

<p align="center">
  <sub>Made with 💛 for the ADHD community</sub>
</p>
