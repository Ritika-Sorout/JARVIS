# JARVIS

A local-first, modular voice assistant. JARVIS listens continuously, transcribes speech, detects when it's being addressed, and answers using a local LLM (via Ollama or any OpenAI-compatible server) with tool-use, persistent memory, and text-to-speech — all running on your own machine.

> **Note on this README:** the repository currently ships only the `jarvis/` Python package (no `README.md`, `requirements.txt`, or `LICENSE` yet). This document was written by reading the source, so it describes what the code actually does rather than a wish list. See [Dependencies](#dependencies) and [License](#license) below for what's missing and what you'll need to add yourself.

## How it works

```
Microphone
    │
    ▼
Voice Activity Detection (webrtcvad)
    │
    ▼
Whisper transcription (faster-whisper / MLX on Apple Silicon)
    │
    ▼
Rolling transcript buffer (~2 min, timestamped)
    │
    ▼  (on wake word / fuzzy match)
Intent Judge (fast-tier LLM)  → directed? query? stop? confidence
    │
    ▼
Planner → Tool Router → Reply Engine (agentic tool-use loop)
    │
    ▼
Text-to-Speech (Piper / Chatterbox) + Dialogue Memory update
```

Rather than a rigid "wake word → single command" pipeline, JARVIS transcribes ambient speech continuously into a rolling buffer and uses a small, fast LLM ("the intent judge") to decide whether a given utterance was actually directed at it — including recognizing fuzzy variants of the wake word ("jarviss", "jervis", "chavis", etc.) and filtering out its own TTS output so it doesn't respond to itself.

## Features

- **Continuous listening, not push-to-talk** — VAD-gated audio → Whisper transcription → an LLM intent judge that separates "talking to Jarvis" from "talking near Jarvis," using conversational context rather than wake-word matching alone.
- **Pluggable LLM backends** — runs against **Ollama** (default, `127.0.0.1:11434`) or any **OpenAI-compatible** server (LM Studio, llama.cpp's `llama-server`, vLLM, LocalAI). A two-tier model system (`Tier.FAST` for quick classification passes, `Tier.CHAT` for the main reply) keeps latency down without sacrificing reply quality.
- **Agentic reply loop** — a pre-flight planner breaks a query into sub-tasks, a tool router selects which tools are relevant, and an evaluator can push the loop to keep working instead of settling for a shallow answer (all of this scales itself down automatically for small local models).
- **Built-in tools**: web search (DuckDuckGo, with Brave Search and a Wikipedia fallback), web page fetching, weather (Open-Meteo, no API key), screenshot + OCR, local file operations, meal/nutrition logging, and a `stop` command for ending a turn gracefully.
- **External tool support via MCP** — connect Model Context Protocol servers and JARVIS will discover and route to their tools alongside the built-ins.
- **Persistent memory** — a SQLite-backed conversation history, a "diary" summarization pass, and a graph-based memory store, with a deterministic recall gate that skips expensive memory lookups when the current context already covers the topic.
- **Dictation mode** — a hold-to-dictate hotkey (Wispr Flow–style) for direct speech-to-text into whatever app has focus, with filler-word removal and a custom dictionary.
- **Configurable text-to-speech** — Piper (default, auto-downloading neural voice) or Chatterbox (with voice cloning via an audio prompt).
- **Persona** — a single adaptive system prompt gives JARVIS a dry, understated "British butler" voice (configurable name, since it just mirrors your chosen wake word) that answers first and, for light topics only, adds one short observation — never for serious or emotional topics.
- **Privacy-oriented defaults** — the default LLM endpoint is `localhost`, nothing is sent off-device unless you point it at a remote server yourself, and inputs are redacted before logging.

## Project layout

```
jarvis/
├── daemon.py              # Orchestrator: wires up listening, memory, TTS, tools
├── main.py                # Entry point (python -m jarvis[.main])
├── config.py              # Settings dataclass, defaults, JSON config + migrations
├── system_prompt.py       # The persona prompt
├── listening/             # VAD, wake-word matching, Whisper transcription, intent judge
├── reply/                 # Planner, tool router, evaluator, the reply engine itself
├── llm/                   # Backend abstraction: Ollama / OpenAI-compatible, model tiers
├── memory/                # Conversation history, diary, graph memory, recall gate
├── tools/                 # Built-in tools + MCP client/registry
├── output/                # TTS engines (Piper / Chatterbox) and audio playback
├── dictation/             # Hold-to-dictate engine and history
└── utils/                 # Vector store, location/IP lookup, fuzzy search, etc.
```

Several subpackages include a `*.spec.md` file (e.g. `llm/llm.spec.md`, `reply/reply.spec.md`, `listening/listening.spec.md`) — these are the most detailed design docs in the repo and are worth reading before making changes to that area.

Note: `listening/listener.py` and `listening/state_manager.py` optionally import a `desktop_app.face_widget` module for an animated face UI. That desktop app isn't part of this repository — JARVIS runs fully headless without it.

## Dependencies

There's no `requirements.txt` or `pyproject.toml` in the repo yet, so install the following manually. Grouped by whether the code always needs them or only needs them for a specific feature:

**Core (always required)**
```
pip install faster-whisper numpy rapidfuzz requests python-dotenv webrtcvad mcp anyio
```

**Feature-specific (install what you plan to use)**
| Feature | Package(s) |
|---|---|
| Wake-word / dictation hotkeys | `pynput` |
| Microphone capture | `sounddevice` |
| Piper TTS (default) | `piper-tts` |
| Chatterbox TTS (voice cloning) | `chatterbox-tts`, `torch`, `torchaudio` |
| Screenshot OCR | `Pillow`, `pytesseract` (+ the Tesseract binary) |
| Fast Apple Silicon transcription | `mlx-whisper` |
| Web page parsing | `beautifulsoup4` |
| Vector memory search | `faiss-cpu` (or GPU build) |
| IP-based location | `geoip2` |
| Notification sound playback | `pygame` |

A local **Ollama** install (or any OpenAI-compatible inference server) is required to actually run the LLM — see [ollama.com](https://ollama.com).

## Getting started

1. Install [Ollama](https://ollama.com) and pull a supported model:
   ```
   ollama pull gemma4:e2b   # default; see config.py for other supported models
   ```
2. Install the Python dependencies listed above.
3. Clone the repo and run:
   ```
   git clone https://github.com/Ritika-Sorout/JARVIS.git
   cd JARVIS
   python -m jarvis.main
   ```
4. Say "Jarvis" followed by your question. Configuration is stored at `~/.config/jarvis/config.json` (created on first run) — see `config.py`'s `get_default_config()` for every available setting: wake word and aliases, TTS engine and voice, Whisper model/backend, memory and tool-routing behavior, and MCP server definitions.

For debugging, `python -m jarvis.main --smoke-test` runs a smoke test, and setting `JARVIS_VOICE_DEBUG=1` enables verbose voice-pipeline logging.

## Configuration highlights

- `llm_provider`: `"ollama"` (default) or `"openai_compatible"`.
- `wake_word` / `wake_aliases` / `wake_fuzzy_ratio`: control what counts as "Jarvis" being addressed.
- `tts_engine`: `"piper"` (default) or `"chatterbox"`.
- `tool_selection_strategy`: `"all"`, `"keyword"`, `"embedding"`, or `"llm"` — how tools are routed per query.
- `mcps`: a dict of external MCP server definitions JARVIS will connect to at startup.

Full defaults live in `jarvis/config.py`.

## License

No `LICENSE` file is currently included in this repository. If you intend for others to use or contribute to this project, consider adding one (e.g. MIT, Apache-2.0).

## Contributing

There's no `CONTRIBUTING.md` yet — if you'd like to accept contributions, the `*.spec.md` files in each subpackage are a good on-ramp for describing expected behavior before opening a PR.
