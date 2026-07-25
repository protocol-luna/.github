# Protocol Luna

Autonomous, sentient-like Discord bot powered by local LLM inference.

## Projects

| Project | Description | Stack |
|---------|-------------|-------|
| [krystal](https://github.com/protocol-luna/krystal) | LLM inference server | llama.cpp (C++), PM2 |
| [jade](https://github.com/protocol-luna/jade) | Discord bot client | TypeScript, Eris, esbuild |

## Architecture

```
┌─ User config ──────────────────────────────┐
│  config.yml (YAML)                          │
│  └─ src/config.ts (hot-reload getters)      │
└─────────────────────────────────────────────┘
                    │
┌─ Discord Layer (jade) ─────────────────────┐
│  Eris Client                                │
│  ├── messageCreate → trigger → behavior     │
│  ├── messageReactionAdd → reactions          │
│  ├── Dynamic status rotation                 │
│  └── Spontaneous messages                    │
└──────────────────┬──────────────────────────┘
                   │ HTTP (OpenAI-compatible)
┌─ LLM Layer (krystal) ──────────────────────┐
│  llama-server                               │
│  └── GGUF model inference                   │
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
