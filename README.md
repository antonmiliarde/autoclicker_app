# 🖱️ Vorozheikin AutoClicker — Release 3

**A production Windows desktop RPA utility that records real user input and replays it with pixel-accurate timing — built as a single-threaded UI + background worker architecture with cloud-backed licensing.**

Stop repeating the same clicks, drags, and keystrokes in ERP forms, browsers, and legacy desktop apps. Record once, edit the step timeline, and replay macros on demand or on a schedule — without writing scripts by hand.

> **Portfolio note:** This is a **rule-based desktop automation** product (event capture → JSON macro DSL → deterministic playback). It does **not** use ML inference, LLMs, or computer-vision models in the application runtime. The engineering focus is **low-level OS integration**, **reliable input injection**, and **full-stack delivery** (desktop client + REST licensing API).

---

## ✨ Key Features

### Free tier (Basic / guest)

| Area | Capability |
|------|------------|
| 🎬 **Recording** | Global mouse listeners (`mouse_down` / `mouse_up` / `move` / `scroll`) and keyboard listeners (`key`, `hotkey`, layout changes) with wall-clock `delay` events between actions |
| ▶️ **Playback** | Configurable repeat count, interruptible delays, PyAutoGUI failsafe (corner abort) |
| 📁 **Macros** | Named macros persisted to `saved_macros.json`; up to **5 macros** on Basic |
| 📋 **Step panel** | View macro steps in a `tksheet` grid; reorder rows; coordinate pickers |
| ⌨️ **Global hotkeys** | Record / stop / play / panic-stop bindings (view-only on Basic when logged in) |
| 🌍 **i18n** | UI in **7 languages**: RU, EN, ZH, FR, DE, PT, ES |
| 🔐 **Account** | Optional sign-in via IT Cabinet Hub (email + password); app remains usable without login |

### Pro / Max tier (paid subscription)

| Area | Capability |
|------|------------|
| ♾️ **Macros** | Unlimited macro library |
| ✏️ **Step editor** | Full spreadsheet editing (cut/paste, inline edits, text steps, random pause insertion) |
| ⚡ **Playback speed** | Per-macro speed multiplier (Basic locked to `1.00×`) |
| 🖱️ **Double-click** | OS-aware double-click detection (`GetDoubleClickTime` / distance metrics) and WinAPI playback |
| 🎯 **Filters** | Column filters + up to 10 saved filter presets per macro |
| ⌨️ **Hotkey presets** | Up to 5 global hotkey configuration presets; editable service hotkeys |
| 🔁 **LMB spam mode** | Toggle infinite left-click automation with configurable interval |
| 🕘 **Macro versions** | Snapshot history with rollback |
| ⏰ **Scheduler** | One-shot and recurring runs (weekdays, month days, every *N* days) with preview |
| 🚀 **Macro activation** | 50 assignable shortcuts (Ctrl/Alt/digit/F-key combinations) |
| 🖥️ **Multi-monitor** | Records virtual-screen viewport; remaps coordinates on playback if monitor layout changed |
| ⌨️ **Keyboard layout** | Captures Windows `HKL` during recording for Cyrillic / Latin playback fidelity |

### Platform integrations

- 🪟 **Windows shell**: system tray icon, Jump List tasks (`sync_hotkey_jump_list.ps1`), single-instance mutex, per-monitor DPI awareness
- ☁️ **Cloud API**: JWT + cookie session, HWID binding, offline grace file (`.vorozheikin_grace` with HMAC)
- 💳 **Payments**: YooKassa checkout via Hub (`POST /payments/yookassa/create/autoclicker`)

---

## 🧱 Tech Stack & Architecture

### Runtime dependencies (`Technical/requirements.txt`)

| Layer | Technology |
|-------|------------|
| Language | **Python 3.8+** (3.10+ recommended for builds) |
| Desktop UI | **Tkinter** + **ttk**, custom styling, **tksheet** spreadsheet widget |
| Input capture | **pynput** (global mouse/keyboard hooks) |
| Input playback | **pyautogui**, **pydirectinput** (DirectX / fullscreen games), **WinAPI** `SendInput` / `mouse_event` via **ctypes** |
| Imaging (UI only) | **Pillow** — icons, tray bitmaps, logo resizing (no CV pipeline) |
| Persistence | **JSON** macros + UI state on disk |
| Networking | **urllib** (stdlib) — HTTPS Hub API, cookie jar, Bearer JWT |
| Concurrency | **threading** — playback worker, LMB spam worker, background HWID verify; **Tk `after()`** for UI scheduling |
| Build / ship | **PyInstaller 6+** (one-file or one-dir), **Inno Setup 6** installer |

### Repository layout

```
.
├── START FROM HERE.bat          # End-user launcher (finds Technical/ by autoclick_recorder.py)
├── README.md
└── Technical/                   # Cyrillic folder name in current tree: Техническая/
    ├── autoclick_recorder.py    # ~22k LOC monolith: UI, engine, licensing, i18n
    ├── launch_app.py            # Dev bootstrap: pip install + pythonw spawn
    ├── launch_hidden.vbs        # Silent launcher for .bat
    ├── saved_macros.json        # Macro + ui_state persistence
    ├── requirements.txt
    ├── file_version_info.txt    # Windows EXE version resource (3.0.0.0)
    └── exe/                     # Build toolchain (PyInstaller, Inno Setup)
        ├── run_pyinstaller.py
        ├── BUILD_CLIENT.bat
        └── dist/                # VorozheikinAutoClicker.exe (after build)
```

### Architecture (high level)

```
┌─────────────────────────────────────────────────────────────┐
│  Tkinter UI (AutoRecorderApp + MacroStepsPanel)             │
│  macro list · step grid · account header · settings         │
└───────────────┬─────────────────────────────┬───────────────┘
                │                             │
     pynput listeners                  threading.Event
     (record thread)                   (playback thread)
                │                             │
                ▼                             ▼
┌───────────────────────────┐   ┌─────────────────────────────┐
│  Event log (JSON list)    │   │  Playback dispatcher        │
│  delay·click·double_click │   │  remap XY · SendInput stack │
│  mouse_*·key·hotkey·text  │   │  coalesce · speed factor    │
└───────────────────────────┘   └─────────────────────────────┘
                │                             │
                └──────────────┬──────────────┘
                               ▼
                    saved_macros.json + Hub API
```

### ML / AI scope (explicit)

| Present in codebase | Not present |
|---------------------|-------------|
| Deterministic event coalescing (double-click, modifier+arrow hotkeys) | Training or inference pipelines |
| HWID fingerprinting (WMI / MachineGuid → SHA-256) | LLM / OCR / template matching |
| HMAC-signed offline grace tokens | PyTorch, TensorFlow, scikit-learn, OpenCV runtime usage |

If you are evaluating this repo for **ML engineering**, treat it as evidence of **shipping complex desktop software**, **API integration**, and **systems programming** — with a clear boundary where ML could be added later (e.g. vision-based step finding).

---

## ⚙️ How It Works (Under the Hood)

### 1. Recording pipeline

1. **pynput** `mouse.Listener` and `keyboard.Listener` run while `_recording` is true.
2. Each physical action appends a typed dict to the active macro list.
3. `_append_delay_if_needed()` inserts `{"type":"delay","seconds":Δt}` from `time.perf_counter()` deltas — preserving real human timing.
4. Mouse drags are represented as `mouse_down` → `mouse_move`* → `mouse_up`.
5. On stop, `coalesce_double_click_events()` merges rapid click pairs into `double_click` using Windows `GetDoubleClickTime()` and `SM_CXDOUBLECLK` / `SM_CYDOUBLECLK`.

**Supported event types:** `delay`, `note`, `click`, `double_click`, `mouse_down`, `mouse_up`, `mouse_move`, `mouse_move_group`, `scroll`, `scroll_group`, `key`, `hotkey`, `text`, `keyboard_layout`.

### 2. Playback pipeline

1. `start_playback()` snapshots events, applies `_coalesce_modifier_nav_key_events()` and `coalesce_double_click_events()`, then spawns a **daemon thread**.
2. `_playback_dispatch_event()` walks the list; delays are **interruptible** (panic hotkey / Esc sets `_stop_playback`).
3. Screen coordinates pass through `remap_screen_xy_for_playback()` when `macro_meta.recording_viewport` differs from the current virtual desktop.
4. Input injection uses a **cascading fallback**:
   - `play_mouse_button_windows()` → `SendInput` / `mouse_event`
   - `play_mouse_double_click_windows()` → WinAPI double-tap, then pyautogui
   - `play_hotkey_windows()` → `SendInput` with extended-key flags
   - `pydirectinput` for game-style ASCII keys
   - `pynput` `keyboard.Controller` for Unicode typing with layout awareness

### 3. Step editor model

- Raw events → `build_merged_step_rows()` → editable rows (pause + comment + event).
- `editable_rows_to_events()` round-trips back to flat JSON without losing drag `sequence` internals.
- Partial playback supports **from row** or **selected rows** subsets.

### 4. Licensing & session (IT Cabinet Hub)

| Endpoint | Purpose |
|----------|---------|
| `POST /hub/login` | Email/password + `app=autoclicker` + HWID → JWT + `it_cabinet_session` cookie |
| `GET /hub/license/status?app=autoclicker` | Tariff (`basic` / `pro`), expiry, `computer_id` |
| `POST /hub/license/hwid-verify` | Binds session to machine fingerprint |
| `POST /hub/payments/yookassa/create/autoclicker` | Subscription checkout URL |

Client-side edition gating (`EDITION_BASIC` vs `EDITION_MAX`) maps Hub tariff to feature flags via `_edition_has_feature()`. Local session continuity uses `.vorozheikin_grace` (HMAC keyed by HWID).

### 5. Packaging

- `exe/run_pyinstaller.py` bundles `autoclick_recorder.py` with hidden imports and brand assets (`app_icon.ico`, `logo.png`, `brand_menu_icons/`).
- Unicode path safe launcher avoids Cyrillic breakage in raw `.bat` → PyInstaller invocations.
- Optional `VorozheikinAutoClicker_Setup.exe` via Inno Setup (`--onedir` payload).

---

## 🛠️ Installation & Local Setup

**Requirements:** Windows 10/11, Python **3.8+** (64-bit recommended), internet for first `pip install`.

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/vorozheikin-autoclicker.git
cd vorozheikin-autoclicker
```

> The source folder is named `Техническая` in the current tree. For a cleaner GitHub layout you may rename it to `Technical` and update `START FROM HERE.bat` accordingly.

### 2. Create and activate a virtual environment

```powershell
cd Technical   # or Техническая
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 3. Install dependencies

```powershell
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Installed packages: `pynput`, `pyautogui`, `tksheet`, `pydirectinput`, `pillow`.

### 4. Run from source

**Option A — launcher script (auto-installs deps):**

```powershell
python launch_app.py
```

**Option B — direct entry point:**

```powershell
python autoclick_recorder.py
```

**Option C — root batch file (no console flash):**

```bat
"START FROM HERE.bat"
```

Logs on failure: `Technical/launch_app.log`, `Technical/startup_crash.log`.

### 5. Build a standalone EXE (developers)

```powershell
cd Technical\exe
pip install -r requirements-build.txt
python run_pyinstaller.py
```

Output: `Technical/exe/dist/VorozheikinAutoClicker.exe`

For an installer with desktop shortcut:

```bat
Technical\exe\BUILD_SETUP.bat
```

→ `Technical/exe/dist_installer/VorozheikinAutoClicker_Setup.exe` (requires [Inno Setup 6](https://jrsoftware.org/isdl.php)).

---

## 📦 How to Run the Pre-compiled Binary

For end users who do not need Python:

1. Open the **[Releases](https://github.com/<your-username>/vorozheikin-autoclicker/releases)** page on GitHub.
2. Download **`VorozheikinAutoClicker.exe`** (portable one-file build) **or** **`VorozheikinAutoClicker_Setup.exe`** (recommended installer).
3. **Portable:** place the `.exe` in any folder and double-click it. No Python install required.
4. **Installer:** run Setup, choose install directory (default: `%LocalAppData%\Programs\VorozheikinAutoClicker`), optionally create a desktop shortcut.
5. If Windows SmartScreen appears (*unsigned binary*), click **More info** → **Run anyway**.

### Runtime data (created next to the executable)

| File | Purpose |
|------|---------|
| `saved_macros.json` | Macros and UI preferences |
| `vorozheikin_account_cookies.txt` | Session cookies |
| `vorozheikin_access_token.txt` | JWT for Hub API |
| `.vorozheikin_grace` | Offline session grace state |

### Unlocking Pro / Max features

Sign in via **Account** in the app header. Paid plans are validated against the IT Cabinet Hub backend (`is_pro_active` / tariff fields). Purchase or renewal opens the web personal cabinet at [itsoulutions.ru/general_enter](https://itsoulutions.ru/general_enter).

---

## 📄 License

**Freemium / proprietary — not open source.**

- The **Basic** edition is free to use with the limitations described above.
- **Pro / Max** functionality requires an authenticated subscription tied to a HWID-bound account.
- Source is published for **portfolio and transparency**; redistribution, modification, and commercial use of the codebase are **not granted** unless explicitly agreed with the author.

`file_version_info.txt` lists **VOROZHEIKIN & COMPANY** as the publisher. No SPDX license file is included in the repository.

For licensing inquiries: [itsoulutions.ru]([https://itsoulutions.ru/](https://itsoulutions.ru/vorozheikinithub/))

---

## 👤 Author
**Anton Vorozheikin** — desktop automation & 1C/ERP tooling  
🔗 https://itsoulutions.ru/vorozheikinithub/ · Portfolio project demonstrating full-stack Windows client delivery

---

<p align="center">
  <sub>Release 3 · Product version 3.0.0.0 · Windows desktop automation</sub>
</p>
