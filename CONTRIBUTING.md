# Contributing to ADHDHolder

Thank you for considering contributing! This project is built with empathy for ADHD users at its core — every PR should reflect that.

## Code of Conduct

- Be respectful and inclusive
- Focus on what's best for the user (ADHD individuals)
- Zero-judgment applies to contributors too — ask questions freely

## How to Contribute

### 1. Setup

```bash
# Prerequisites: Node.js 20+, Rust 1.75+, and a system with WebView2
git clone https://github.com/your-org/adhdholder
cd adhdholder
npm install
cd src-tauri && cargo build && cd ..
```

### 2. Development

```bash
# Frontend dev (hot reload in browser)
npm run dev

# Full Tauri dev (desktop window + hot reload)
npm run tauri:dev
```

### 3. Design Principles

Before writing code, read `docs/DESIGN.md` — it outlines the three non-negotiable constraints:

1. **Course set by human, not guessed by machine** — never infer what the user is doing from input data
2. **Perception is rhythm, not content** — never read keystrokes, window titles, or screen content
3. **Start with the lightest intervention** — false alarms are themselves a disturbance

### 4. Pull Request Process

1. Keep changes focused — one feature/fix per PR
2. Update `CHANGELOG.md` with your change
3. Ensure the app builds: `npx tauri build --no-bundle`
4. Test in both Tauri and browser (dev) modes
5. Make sure audio, input detection, and the state machine still work

### 5. Getting Help

Open an issue for bugs, feature requests, or questions. No question is too small.
