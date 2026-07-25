# Protocol Luna

Autonomous, sentient-like bots powered by local LLM inference. Discord (jade) and Matrix (pixieglow).

## Projects

| Project | Description | Stack |
|---------|-------------|-------|
| [sapphire](https://github.com/protocol-luna/sapphire) | LLM gateway (classification + routing) | Python, fastembed, FastAPI |
| [krystal](https://github.com/protocol-luna/krystal) | LLM inference server (small/large mode) | llama.cpp (C++), PM2 |
| [jade](https://github.com/protocol-luna/jade) | Discord bot client | TypeScript, Eris, esbuild |
| [pixieglow](https://github.com/protocol-luna/pixieglow) | Matrix bot | TypeScript, Bun |

## Architecture

```
┌─ User config ──────────────────────────────┐
│  config.yml (YAML)                          │
│  └─ src/config.ts (hot-reload getters)      │
└─────────────────────────────────────────────┘
                    │
┌─ Bot Layer ────────────────────────────────┐
│  Jade (Discord) ──┐                        │
│  Pixieglow (Matrix)┘                       │
│  └── OpenAI-compatible HTTP                 │
└──────────────────┬──────────────────────────┘
                   │
┌─ Gateway Layer (sapphire) ─────────────────┐
│  Classifies: FUTILE vs INTERESSANT          │
│  (embedding centroid, 99.6% accuracy)       │
│  Routes to appropriate Krystal instance     │
└──────────────────┬──────────────────────────┘
                   │
┌─ LLM Layer (krystal) ──────────────────────┐
│  Krystal-small (port 3124)                  │
│  └── Luna-Protocol-1.5B (fast, GENERIC)     │
│  Krystal-large (port 3125)                  │
│  └── Discord-Hermes-8B (deep, SEMANTIC)     │
└─────────────────────────────────────────────┘
```

## Behaviors

- Variable concentration (per-trigger delays)
- Keyboard typos (AZERTY / QWERTY) with correction
- Message bursts (2-3 fragments at human pace)
- Topic fatigue detection
- Hesitation words, forgetfulness
- Inactivity warmup
- Sleep schedules (time-based)
- Voice messages via Piper TTS
- Spontaneous unprompted messages
- Dynamic Discord presence rotation

## Documentation

Detailed diagrams, state machines, and explanations are available in the [`state-machines/`](state-machines/) folder — 24 Mermaid diagrams covering the entire codebase.

## Models

Fine-tuned LLM models for Discord conversation:

- [Luna-Protocol-1.5B (Qwen2.5)](https://huggingface.co/fox3000foxy/Luna-Protocol-1.5B-Discord-Dialogues-50k-instruct) — Recommended
- [Dataset: Discord-Dialogues](https://huggingface.co/datasets/mookiezi/Discord-Dialogues) — 7.3M exchanges
