# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MoneyPrinterV2 (MPV2) is a Python 3.12 CLI tool that automates four online workflows:
1. **YouTube Shorts** — generate video (LLM script → TTS → images → MoviePy composite) and upload via Selenium
2. **Twitter/X Bot** — generate and post tweets via Selenium
3. **Affiliate Marketing** — scrape Amazon product info, generate pitch, share on Twitter
4. **Local Business Outreach** — scrape Google Maps (Go binary), extract emails, send cold outreach via SMTP

There is no web UI, no REST API, no test suite, no CI, and no linting config.

## Running the Application

```bash
# First-time setup
cp config.example.json config.json   # then fill in values
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# macOS quick setup (auto-configures Ollama, ImageMagick, Firefox profile)
bash scripts/setup_local.sh

# Preflight check (validates services are reachable)
python scripts/preflight_local.py

# Run
python src/main.py
```

The app **must** be run from the project root. `python src/main.py` adds `src/` to `sys.path`, so all imports use bare module names (e.g., `from config import *`, not `from src.config import *`).

## Architecture

### Directory Structure

```
MoneyPrinterV2/
├── src/                    # Application source code
│   ├── main.py             # Interactive menu entry point
│   ├── cron.py             # Headless runner for scheduled jobs
│   ├── config.py           # 30+ getter functions, re-reads config.json each call
│   ├── llm_provider.py     # Ollama SDK wrapper (generate_text, list/select models)
│   ├── cache.py            # JSON file persistence in .mp/ directory
│   ├── constants.py        # Menu strings, Selenium CSS/XPath selectors
│   ├── status.py           # Colored console output helpers (termcolor)
│   ├── utils.py            # Selenium cleanup, temp file removal, song fetching
│   ├── art.py              # ASCII banner printer
│   └── classes/
│       ├── YouTube.py      # Full pipeline: topic → script → images → TTS → subtitles → video → upload (~850 lines)
│       ├── Twitter.py      # Tweet generation + Selenium posting (~226 lines)
│       ├── AFM.py          # Amazon scraping + LLM pitch generation (~177 lines)
│       ├── Outreach.py     # Google Maps scraper (Go) + email sending (~297 lines)
│       └── Tts.py          # KittenTTS wrapper (~19 lines)
├── scripts/
│   ├── setup_local.sh      # One-time bootstrap (venv, deps, config, preflight)
│   ├── preflight_local.py  # Validates config, Ollama, Gemini API, Whisper
│   └── upload_video.sh     # Direct video upload helper
├── docs/                   # Feature documentation (Configuration, YouTube, Twitter, etc.)
├── assets/banner.txt       # ASCII art banner
├── fonts/bold_font.ttf     # Font for video subtitle rendering
├── .mp/                    # Runtime cache + scratch space (gitignored)
├── Songs/                  # Background music archive (auto-downloaded, gitignored)
├── config.example.json     # Configuration template (35 settings)
├── config.json             # User config (gitignored, never commit)
├── requirements.txt        # Python dependencies
├── AGENTS.md               # Guidelines for AI coding agents
└── CONTRIBUTING.md         # PR and contribution guidelines
```

### Entry Points
- `src/main.py` — interactive menu loop (primary). Startup sequence: print banner → create `.mp/` if needed → clean temp files → fetch songs → select Ollama model → enter menu loop.
- `src/cron.py` — headless runner invoked by the scheduler as a subprocess: `python src/cron.py <platform> <account_uuid> [model_name]`. Supports `youtube` and `twitter` platforms.

### Provider Pattern

| Category | Config key | Options |
|---|---|---|
| LLM | `ollama_model` | Ollama (via `ollama` Python SDK). If empty, user picks from available models at startup. |
| Image gen | `nanobanana2_*` | Gemini image API (`gemini-3.1-flash-image-preview` default). Direct HTTP calls, no SDK. |
| TTS | `tts_voice` | KittenTTS (`KittenML/kitten-tts-mini-0.8`). Voices: Bella, Jasper, Luna, Bruno, Rosie, Hugo, Kiki, Leo. |
| STT | `stt_provider` | `local_whisper` (faster-whisper library) or `third_party_assemblyai` |

### Key Modules

- **`src/config.py`** — 30+ getter functions, each re-reads `config.json` on every call (no caching). `ROOT_DIR` = project root, computed as `os.path.dirname(sys.path[0])`. Some keys fall back to environment variables (e.g., `GEMINI_API_KEY` for `nanobanana2_api_key`).
- **`src/llm_provider.py`** — unified `generate_text(prompt)` function using the Ollama Python SDK. Module-level global `_selected_model` for state. Helper functions: `list_models()`, `select_model()`, `get_active_model()`.
- **`src/cache.py`** — JSON file persistence in `.mp/` directory. Separate files: `youtube.json`, `twitter.json`, `afm.json`. CRUD functions for accounts, videos, posts, products.
- **`src/constants.py`** — menu option strings + Selenium selectors for YouTube Studio, X.com, and Amazon.
- **`src/utils.py`** — `close_running_selenium_instances()` (cross-platform Firefox kill), `rem_temp_files()` (removes non-JSON from `.mp/`), `fetch_songs()` (downloads ZIP archive), `choose_random_song()`.

### Class Details

- **`YouTube.py`** — Most complex. Pipeline: `generate_topic()` → `generate_script()` → `generate_metadata()` → `generate_prompts()` → `generate_image_nanobanana2()` (Gemini HTTP API, saves base64 PNG to `.mp/`) → `generate_script_to_speech()` (KittenTTS) → `generate_subtitles()` (Whisper or AssemblyAI, outputs SRT) → `combine()` (MoviePy composites images + audio + subtitles into MP4) → `upload_video()` (Selenium file picker + metadata form).
- **`Twitter.py`** — `post(text=None)` generates tweet via LLM then posts with Selenium. Multiple CSS/XPath fallbacks for X.com UI changes.
- **`AFM.py`** — `scrape_product_information()` extracts title + features from Amazon via Selenium. `generate_pitch()` creates marketing text. `share_pitch()` posts to Twitter.
- **`Outreach.py`** — Downloads external Go scraper (`gosom/google-maps-scraper` v0.9.7) as ZIP, builds with `go build`, runs as subprocess. Extracts emails from scraped business websites via regex. Sends via yagmail SMTP.
- **`Tts.py`** — Thin wrapper: `synthesize(text, output_file)` → KittenTTS generates audio array → writes WAV via `soundfile` at 24kHz.

### Data Storage

All persistent state lives in `.mp/` at the project root as JSON files. This directory also serves as scratch space for temporary WAV, PNG, SRT, and MP4 files — non-JSON files are cleaned on each run by `rem_temp_files()`.

### Browser Automation

Selenium uses pre-authenticated Firefox profiles (never handles login). The profile path is stored per-account in the cache JSON and also in `config.json` as a default. `webdriver_manager` auto-downloads geckodriver. Headless mode is configurable. `WebDriverWait` uses 30-second timeouts.

### CRON Scheduling

Uses Python's `schedule` library (in-process, not OS cron). The scheduled job spawns `subprocess.run(["python", "src/cron.py", platform, account_id, model_name])`.

## Configuration

All config lives in `config.json` at the project root. See `config.example.json` for the full template (35 settings) and `docs/Configuration.md` for reference.

**Key external dependencies to configure:**
- **ImageMagick** — required for MoviePy subtitle rendering (`imagemagick_path`: path to `convert` or `magick.exe`)
- **Firefox profile** — must be pre-logged-in to target platforms (`firefox_profile`)
- **Ollama** — for LLM text generation (`ollama_base_url`, default `http://127.0.0.1:11434`)
- **Gemini API** — for image generation (`nanobanana2_api_key` or `GEMINI_API_KEY` env var)
- **Go** — only needed for Outreach (Google Maps scraper)

**Security:** Never commit `config.json`. It is gitignored. Prefer environment variables where supported.

## Import Conventions

Because `src/` is added to `sys.path`, all imports use bare module names:
```python
# Correct
from config import get_verbose, ROOT_DIR
from llm_provider import generate_text
from cache import get_youtube_accounts

# Wrong — do not use src. prefix
from src.config import get_verbose
```

## Coding Conventions

- **Python 3.12** target
- **4-space indentation**
- **snake_case** for functions/variables, **PascalCase** for classes, **UPPER_SNAKE_CASE** for constants
- Business logic goes in focused modules under `src/`; provider/integration code goes in `src/classes/`
- Prefer small, explicit functions
- Preserve CLI-first behavior (no web UI assumptions)

## Dependencies (requirements.txt)

Core: `ollama`, `selenium`, `selenium_firefox`, `webdriver_manager`, `moviepy`, `Pillow>=10.0.0`, `kittentts` (custom wheel from GitHub), `soundfile`, `faster-whisper`, `assemblyai`, `srt_equalizer`, `yagmail`, `termcolor`, `schedule`, `prettytable`, `platformdirs`, `undetected_chromedriver`.

## Validation

There is no automated test suite. To validate changes:
1. Run `python scripts/preflight_local.py` (checks config, Ollama, Gemini API, Whisper)
2. Smoke-test impacted flows via `python src/main.py`

When adding tests in the future, place them in a top-level `tests/` directory with names like `test_<module>.py`.

## Contributing

PRs go against `main`. One feature/fix per PR. Open an issue first. Use `WIP` label for in-progress PRs. Follow imperative commit style: `Fix ...`, `Update ...`, `Add ...`.
