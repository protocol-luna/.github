# Luna Protocol -- Detailed Overview

Fully autonomous and sentient-like Discord bot. Runs a local LLM (llama.cpp) and converses naturally -- sleep, inattention, typos, hesitations, forgetfulness, topic fatigue, message bursts, voice messages, anti-spam queue, persistence, auto-restart, rotating status.

> **Luna Protocol now has its own official models!**
>
> The project includes optimized models trained specifically for Discord conversations:
> - **Luna-Protocol-1.5B-Fine-Tuned-Qwen2.5** (Recommended) -- Lightweight, fast, perfect for Discord bots
> - Available on [HuggingFace](https://huggingface.co/fox3000foxy/Luna-Protocol-1.5B-Discord-Dialogues-50k-instruct)
>
> These models come with **few-shot priming support** to guide conversation style and improve consistency.

- Fine-tuned Qwen2.5 1.5B model, trained on 50k samples from [Discord-Dialogues](https://huggingface.co/datasets/mookiezi/Discord-Dialogues) (subset of the full 7.3M exchanges dataset)
- Quantized GGUF format
- **Few-shot priming**: Inject example conversations to guide model behavior (configurable in `config.yml`)
- Two LLM modes: `direct` (bot → llama‑server directly, shared model / prompt cache), `online` (OpenAI‑compatible API)
- Event-driven architecture: `llmBus` for LLM tokens/errors, `stateBus` for auto-persist

---

## Trigger System

### State machine -- incoming message decision

### Trigger priority order

| # | Reason | Conditions | Bypass ignore | Bypass pause |
|---|---|---|---|---|
| 1 | `mention` | @bot | Yes (0%) | Yes |
| 2 | `dm` | DM with `replyInDM = true` | Yes (0%) | No |
| 3 | `name` | "Luna"/"Pixie"/alias (whole word) | No (8%) | No |
| 4 | `keyword` | `hello`, `hi`, `hey`, `yo`, `ai`, `bot`... (whole word) | No (8%) | No |
| 5 | `follow-up` | Bot was last speaker + < 15s + < 3 / 60s | -- | -- |
| 6 | `random` | 1.5% chance on non-matching | No (8%) | No |

Whole word matching (`\b`): "ai" does not match "mais", "vrai", "lait".

### Cooldown

8 seconds between two responses in the same channel. Bypassed by mentions and follow-ups.

### Follow-up

The bot registers itself as `lastSpeaker`. Any subsequent message within 15s triggers an immediate response (no timer, no keyword check). Budget: 3 follow-ups per 60s window.

---

## Response Mechanisms

### Variable concentration

| Trigger | Min delay | Max delay | Ignore | Reaction |
|---|---|---|---|---|
| `mention` | 300ms | 1500ms | 0% | 8% |
| `dm` | 400ms | 1800ms | 0% | 5% |
| `name` | 800ms | 4000ms | 5% | 6% |
| `keyword` | 1000ms | 3500ms | 8% | 4% |
| `follow-up` | 500ms | 2000ms | 0% | 3% |
| `random` | 1500ms | 5000ms | 15% | 2% |

Configurable via `concentration` in `config.yml`.

### Typos

Configurable probability (`typo_chance`, default 6%) of replacing a letter with an adjacent key (AZERTY/QWERTY). Correction after 2-4s:

| Style | Behavior |
|---|---|
| `edit` | Edits the message |
| `message` | New message: `word*` |
| `mixed` | 50/50 random (default) |

AZERTY example: `bonjour → bonjpur`, `salut → slaut`, `comment → cpmment`.

### Voice Messages (TTS)

Configurable probability (`voice_message_chance`, default 8%). Full pipeline: sanitize → synthesize (Piper) → convert (ffmpeg) → upload (Discord CDN).

### Typing indicator

`startTyping()` is called before `askLLM()` -- the typing indicator stays active during generation (refreshed every 8s).

### Real-time response

The LLM streams its response line by line (`\n`). Each line is split into words, emitted one by one. No simulated delay: the pace is the LLM's own.

### Reactions

30% server custom emoji, 70% unicode emoji.

### Reply style

Weighted according to recent bot activity in the channel.

### Sleep schedules

| Mode | Effect |
|---|---|
| `sleep` | Only mentions and DMs pass through |
| `slow` | Delay ×3-5, reactions nearly zero |
| `short` | Ignore chance +30%, reactions nearly zero |

### Spontaneous Messages

Every 5 minutes, 12% chance the bot posts a message on its own initiative.

### Hesitation

The bot sometimes starts its response with a hesitation word: `uh...`, `um...`, `well...`, `i mean...`, `hmm...`, `so...`.

### Forgetfulness

Even after matching a trigger, the bot can "forget" to respond with a probability of `forget_chance` (default 3%).

### Inactivity warmup

If the bot has not been active for `inactivity_warmup_minutes` (default 10 min), the response delay is multiplied by `inactivity_warmup_multiplier`.

### Dynamic Discord Status

The Discord status alternates between several configured presets, rotating every `dynamic_status_interval_minutes` minutes.

### Anti-spam

Key `channelId:userId`. Only one message queued per user per channel.

### Persistence

**Persisted:** pendingMessages, paused, cooldowns, timestamps, lastSpeaker, follow-up counters.
**Auto-save:** any state mutation emits on `stateBus` → automatic save (debounce 500ms).

---

## Commands

**By text:** `-stop` (pause + reset), `-start` (resume), `-clear` (reset history)

**By reactions** on one of the bot's messages:

| Emoji | Effect |
|---|---|
| ❌ | Stop |
| ▶️ | Start |
| 🗑️ | Clear |

---

## Configuration

Single `config.yml` file. Shell env vars override YAML keys if present. Hot-reload for dynamic values.

### Hot-reload legend

| Icon | Meaning |
|---|---|
| ✅ | Hot-reloadable -- changes picked up at runtime |
| ❌ | Requires restart |

### Secrets & Paths (❌)

- `discord_token` -- Discord bot token
- `llama_model_path` -- Path to GGUF model
- `llm_host` / `llm_port` -- LLM server connection
- `llm_mode` -- `direct` (local llama-server) or `online` (OpenAI API)
- `tts_model_path`, `tts_binary_path`, `ffmpeg_path`, `ffprobe_path` -- TTS & audio paths

### Triggers (✅)

`names`, `keywords`, `random_chance`, `cooldown_seconds`, `reply_in_dm`

### Mannerisms -- Concentration (✅)

Per-trigger delays, ignore & reaction chances.

### Typos (✅)

`typo_chance`, `typo_layout` (azerty/qwerty), `typo_correction_style` (edit/message/mixed)

### Message Burst (✅)

`burst_chance`, `burst_delay_min`, `burst_delay_max`

### Topic Fatigue (✅)

Detects repetitive topics per channel. Configurable window, threshold, multiplier.

### Human-like Behaviors (✅)

`hesitation_chance`, `hesitation_words`, `forget_chance`, `inactivity_warmup`

### Dynamic Status (✅)

Rotation interval + status presets (Playing, Watching, Listening, etc.)

### Sleep Schedules (✅)

Time-based behavior switching (sleep/slow/short).

### Session Limits (✅)

Max exchanges before pause, pause duration, reset timer.

### TTS / Voice Messages (✅)

Voice message probability.

### Reply Styles (✅)

Weighted message_reference + mention_replied_user configs.

---

## Logs

| Prefix | Info |
|---|---|
| `[trigger]` | Evaluation + result of each message |
| `[mannerisms]` | Delay, ignore, reaction, msgLength, inactivity |
| `[bot]` | Decision, follow-up, reply style, forget |
| `[tts]` | Synthesis, upload, voice message |
| `[persist]` | Save/restore |
| `[llm-core]` | Queue, proxy/online, LLM events |
| `[llmBus]` | LLM events (token, done, flush, error, ready) |

---

## Dataset

[**Discord-Dialogues**](https://huggingface.co/datasets/mookiezi/Discord-Dialogues) -- 7.3M exchanges, 17M turns, 140M words. Real Discord conversations spring-summer 2025, filtered PII/ToS/bots/commands. Apache 2.0.

| Metric | Value |
|---|---|
| Samples | 7 303 464 |
| Total turns | 16 881 010 |
| Total words | 139 922 950 |
| Average tokens | 32.8 |
| Tokenizer | Hermes-3-Llama-3.1-8B |

## Discord Developer Portal

- **Message Content Intent** (Bot tab)
- Scope `bot` + permissions: `Send Messages`, `Read Message History`, `Add Reactions`
- Gateway intents: `guilds`, `guildMessages`, `guildMessageReactions`, `messageContent`, `directMessages`
