# Luna Protocol -- Detailed Overview

Fully autonomous Discord / Matrix bot. Runs a local LLM (llama.cpp) and converses naturally -- sleep, inattention, typos, hesitations, forgetfulness, topic fatigue, message bursts, voice messages, anti-spam queue, persistence, auto-restart, rotating status.

> **Luna Protocol v2 -- Emerald era**
>
> The architecture was reorganized in July 2026: a new brain service (Emerald) now sits between bots and Sapphire. Bots are thin WebSocket clients with no direct Sapphire or LLM logic.

- Fine-tuned Qwen2.5 1.5B model, trained on 200k samples from [Discord-Dialogues](https://huggingface.co/datasets/mookiezi/Discord-Dialogues)
- Quantized GGUF format (Q4_K_M, 941 MB)
- **Few-shot priming**: Sapphire injects example conversations before the system prompt
- **K-means multicentroid classification**: 10 centroids per class (k=10, seed=42), reproducible
- **Emotion-aware sampling**: Temperature, repeat penalty, mirostat entropy adjusted by valence/arousal
- Architecture: Platform &rarr; Bot (WebSocket) &rarr; Emerald (brain) &rarr; Sapphire (HTTP) &rarr; Krystal (llama.cpp)
- All behavior and LLM logic lives in Emerald + Sapphire; bots are adapters only

---

## Architecture

```
Discord / Matrix
     |
     v
  +----------+    +-------------+
  | Jade     |    | Pixieglow   |  <-- Thin adapters (WebSocket clients)
  | (Discord)|    | (Matrix)    |
  +-----+----+    +------+------+
        |                |
        +--------+-------+
                 |
           WebSocket :3126
                 |
          +------v------+
          |   Emerald    |  <-- Brain: behavior, Sapphire streaming, typo/swap, voice decision
          |  (TypeScript) |
          +------+-------+
                 |
            HTTP :3123
                 |
           +------v-------+
           |   Sapphire    |  <-- Gateway: classification, sessions, few-shot, emotion, streaming
           |   (Python)    |
            +---+-------^---+-------------------+
                |       |                       |
                |  FUTILE / INTERESSANT         |
                |       |                  HTTP :3127
           HTTP :3124  HTTP :3125               |
                |       |              +--------v--------+
         +------v---+ +-v--------+     | Ruby            |
         | Krystal   | | Krystal  |     | Markov Chain    |
         | small Q4  | | large    |     | (TypeScript)   |
         | (C++)     | | (C++)    |     +-----------------+
         +-----------+ +----------+
```

### Services

| Service | Language | Port | Role |
|---|---|---|---|
| Emerald | TypeScript | 3126 | Brain: behavior, triggers, streaming, WebSocket server |
| Sapphire | Python | 3123 | LLM gateway: classification, emotion, sessions, few-shot |
| Krystal (generic) | C++ (llama.cpp) | 3124 | Small LLM for FUTILE messages |
| Krystal (semantic) | C++ (llama.cpp) | 3125 | Large LLM for INTERESSANT messages |
| Ruby | TypeScript | 3127 | Markov chain for ambient messages |

---

## Emerald (Brain)

Emerald is the central decision engine. It runs a WebSocket server on port 3126 that platform adapters (Jade, Pixieglow) connect to.

### Streaming flow

1. Bots connect via WebSocket and forward `MessageEvent`s (with optional `debug: true`)
2. Emerald evaluates behavior rules: burst cooldown, sleep schedule, typo/swap probability, voice chance
3. **Mention stripping**: bot mentions (`@Kalupso`, `<@userId>`) are removed from the text
4. Emerald calls Sapphire via **streaming** (`askStream`) -- the response arrives token by token
5. **On the first token**, Emerald sends a `TypingCommand` to the bot -- the typing indicator appears before generation completes
6. Emerald buffers the streaming tokens until the full response is received
7. Sapphire checks for degenerate output; if degenerate, the stream is discarded silently
8. Emerald applies typo/swap behavior to the response text
9. Emerald sends a `RespondCommand` back with `responseText`, optional `voice` flag, and optional `debugStats`
10. The bot posts the response (optionally as a voice message if `voice: true`)

### Ruby integration

Emerald forwards every human message to Ruby's `/train` endpoint (fire-and-forget). For `random` and `spontaneous` trigger types, Emerald calls `ruby.generate()` instead of Sapphire. The bot receives a `RespondCommand` either way -- it never knows whether the response came from an LLM or a Markov chain.

### State isolation

BrainState is isolated per `(client, channel)` composite key, preventing state collisions when multiple adapters (Jade + Pixieglow) share a channel.

### Key files

- `src/server.ts` -- WebSocket server (connection management, event routing, command dispatch)
- `src/brain.ts` -- Decision engine: evaluates behavior, calls Sapphire via streaming, applies typo/swap
- `src/sapphire-client.ts` -- HTTP client with `ask()` (non-streaming) and `askStream()` (SSE streaming)
- `src/protocol.ts` -- WebSocket message types (events, commands, debug stats, behavior debug)
- `src/behavior/*.ts` -- Burst, sleep, typo, mannerisms
- `src/state/*.ts` -- State management, triggers, topic fatigue

---

## Trigger System

### Trigger priority order

Emerald's `evaluateMessage()` checks conditions in this exact order:

| # | Reason | Conditions | Bypass pause |
|---|---|---|---|
| 1 | `mention` | `event.mentions` includes bot userId or username | Yes |
| 2 | `dm` | DM with `config.reply_in_dm = true` | No |
| 3 | `name` | Text contains bot username or `config.names` entries | No |
| 4 | `keyword` | Matches `config.keywords` (probabilistic via `keyword_chance`) | No |
| 5 | `follow-up` | Bot was last speaker + within follow-up window + max 3 replies | -- |
| 6 | `random` | `config.random_chance` probability | No |

Special commands:
- `-stop` -- Pause + reset for session
- `-start` -- Resume session
- `-clear` -- Reset conversation history

### Cooldown

`config.cooldown_seconds` (default 8s) between two responses in the same channel. Bypassed by mentions.

### Follow-up

The bot registers itself as `lastSpeaker`. Any subsequent message within `follow_up_window` (default 15s) triggers an immediate response. Budget: max 3 follow-ups per 60s window.

---

## Behavior System

All behaviors are evaluated by Emerald. The bot itself has no behavior logic.

### Attention / Concentration

Emerald computes dynamic delay, ignore chance, and reaction probability per trigger:

| Trigger | Min delay | Max delay | Ignore | Reaction |
|---|---|---|---|---|
| `mention` | 300ms | 1500ms | 0% | 8% |
| `dm` | 400ms | 1800ms | 0% | 5% |
| `name` | 800ms | 4000ms | 5% | 6% |
| `keyword` | 1000ms | 3500ms | 8% | 4% |
| `follow-up` | 500ms | 2000ms | 0% | 3% |
| `random` | 1500ms | 5000ms | 15% | 2% |

Configurable via `concentration` in Emerald's `config.yml`.

### Burst

When engaged in an active conversation, Emerald splits long responses into fragments sent with delays. Configurable probability (default 15%) and delay range (1.5-4s between fragments).

### Sleep Schedule

| Mode | Effect |
|---|---|
| `sleep` | Only mentions pass through; all other messages ignored |
| `slow` | Response delay multiplied by 3-5x |
| `short` | Ignore chance increased by 30% |

### Typos / Letter-Swap

Applied to the response text after receiving it from Sapphire. Typo chance replaces a letter with an adjacent key (AZERTY or QWERTY). Swap chance transposes adjacent letters. Corrections are sent as edits with configurable delay (2-4s).

### Hesitation Markers

Configurable chance of injecting a hesitation word (`uh...`, `hmm...`, `well...`) at the start of the response.

### Topic Fatigue

Tracks word frequency per channel. When a topic repeats too often (default threshold: X mentions in last Y messages), response delay doubles and ignore chance increases. Resets after Y new messages.

### Voice Messages

Emerald decides whether the response should be sent as a voice message (configurable chance). If `voice: true`, the bot (Jade) generates Piper TTS audio and uploads it as a Discord voice message.

### Forgetting

Configurable chance that Emerald returns a `forgot` command instead of responding.

### Spontaneous Messages

Every 5 minutes, configurable chance the bot posts a message on its own initiative. Handled by Ruby (Markov chain) to avoid LLM costs.

### Inactivity Warmup

If the bot has not been active for `inactivity_warmup_minutes` (default 10 min), the response delay is multiplied by `inactivity_warmup_multiplier`.

### Dynamic Discord Status

The Discord status alternates between configured presets, rotating every `dynamic_status_interval_minutes` minutes.

### Anti-spam

Key `channelId:userId`. Only one message queued per user per channel.

### Persistence

**Persisted:** pendingMessages, paused, cooldowns, timestamps, lastSpeaker, follow-up counters.
**Auto-save:** any state mutation emits on stateBus &rarr; automatic save (debounce 500ms).

---

## Sapphire (LLM Gateway)

Sapphire is a Python FastAPI server on port 3123.

### Classification

Messages are embedded via fastembed (`BAAI/bge-small-en-v1.5`, 384-dim). **K-means multicentroid** (k=10, seed=42, max 20 iters) computes 10 centroids per class from example phrases. Classification uses maximum cosine similarity across all centroids:

```
sim_futile = max(cos_sim(emb, c) for c in futile_centroids)
sim_interessant = max(cos_sim(emb, c) for c in interessant_centroids)
delta = sim_interessant - sim_futile
label = INTERESSANT if delta > 0 else FUTILE
confidence = abs(delta)
```

The fixed seed (42) ensures reproducible centroids across rebuilds. When fewer than `k` examples exist, a single mean centroid is used instead.

### Emotion Scoring

Four emotion centroids (positive, negative, high_arousal, low_arousal), each with k-means multicentroid (k=10). Valence and arousal computed on [-1, 1] scale:

```
valence = cos_sim(emb, positive) - cos_sim(emb, negative)
arousal = cos_sim(emb, high_arousal) - cos_sim(emb, low_arousal)
```

Emotion state per conversation decays via EMA (decay factor 0.85). Deadzone: 0.005 (per-message scores below threshold don't update the running state).

### Emotion-Aware Sampling

Sapphire dynamically adjusts Krystal's sampling parameters based on emotional state:

| Parameter | Formula | Effect |
|---|---|---|
| Temperature | `clamp(0.7 + arousal × 0.3, 0.4, 1.0)` | Higher arousal = more randomness |
| Repeat penalty | `clamp(1.15 - valence × 0.1, 1.0, 1.3)` | Positive mood = less penalty |
| Mirostat entropy | `clamp(6.0 + arousal × 2.0, 3.0, 8.0)` | Higher arousal = more entropy |

### Session Management

Per-channel conversation history with configurable max messages (default 20) and TTL (default 600s).

### Few-Shot Injection

Example conversations from `few_shot_examples.yml` injected server-side after the system prompt:

```python
# In Sapphire's few_shot.py:
def inject_few_shot_into_conversation(messages, few_shot_messages):
    system, *rest = messages
    return [system, *few_shot_messages, *rest]
```

Example file format:

```yaml
# few_shot_examples.yml
- user: "yo whats good"
  assistant: "nm just chillin, u"
- user: "bored af"
  assistant: "lol same energy fr"
```

### Degenerate Detection

Regex-based detection of degenerate outputs (empty, repetitive, whitespace-only). Non-streaming: retry up to `llm_max_retries` (default 2). Streaming: silently discard and return fallback.

### Streaming SSE

Returns an SSE stream of content tokens, then final metadata JSON, then `[DONE]`. Format:

```
data: Hello
data: !
data:  How
data: ...
data: {"text":"Hello! How...","label":"FUTILE","backend":"...","valence":0.12,"arousal":-0.04}
data: [DONE]
```

### Backend Routing

| Label | Backend |
|---|---|
| FUTILE | `krystal_generic_url` (:3124) |
| INTERESSANT | `krystal_semantic_url` (:3125) |

Set both to the same URL for single-backend mode.

### Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/v1/respond` | POST | Classify + session + few-shot + emotion + generate. Main pipeline. |
| `/v1/chat/completions` | POST | OpenAI-compatible chat completions (no session, no few-shot) |
| `/v1/reset` | POST | Reset a conversation session |
| `/classify` | POST | Classify text without generation |
| `/emotion/{key}` | GET | Get emotional state for a session |
| `/health` | GET | Server health check with config summary |

### Key files

- `src/sapphire/server.py` -- FastAPI server, endpoints, streaming
- `src/sapphire/classifier.py` -- K-means multicentroid, classification
- `src/sapphire/emotion.py` -- Emotion scoring, EMA state, multicentroid
- `src/sapphire/few_shot.py` -- Few-shot loading and injection
- `src/sapphire/sessions.py` -- Session store with TTL
- `src/sapphire/proxy.py` -- HTTP client for backend LLM calls
- `src/sapphire/degenerate.py` -- Degenerate output detection

---

## Krystal (LLM Server)

Krystal runs `llama-server` from llama.cpp. Sapphire routes FUTILE to the generic backend and INTERESSANT to the semantic backend.

Current model: `Luna-Protocol-1.5B-Fine-Tuned-Qwen2.5-200k-instruct.Q4_K_M.gguf` (941 MB).

| Parameter | Value |
|---|---|
| Port | 3124 / 3125 |
| Context | 4096 tokens |
| GPU layers | 0 (CPU only) |
| Threads | 6 |
| Batch size | 1024 |
| Cache reuse | Enabled (--cache-reuse) |

---

## Ruby (Markov Chain)

Ruby is a TypeScript Markov chain service running on port 3127. It handles ambient messages without any LLM call.

### Role

Ruby handles `random` and `spontaneous` trigger types. For these, Emerald calls `ruby.generate()` instead of Sapphire -- zero GPU/VRAM usage, ~1ms latency.

### How it works

- Emerald forwards every human message to Ruby's `/train` endpoint (fire-and-forget)
- Ruby tokenizes messages and builds an order-2 Markov chain in SQLite via `better-sqlite3` (WAL mode)
- Each 2-word prefix maps to possible next words with frequency counts
- Generation uses weighted random selection from frequency table
- Database is immediately durable (WAL mode) -- no periodic snapshots

### Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/train` | POST | Train on a message |
| `/train-batch` | POST | Train multiple messages atomically |
| `/generate` | POST | Generate text with optional seed and max length |
| `/channels` | GET | List known channel IDs |
| `/stats` | GET | Transition/starter counts and DB size |
| `/health` | GET | Health check |

### Key files

- `src/server.ts` -- Express server with all endpoints
- `src/chain.ts` -- Markov chain implementation (order-2, SQLite backends)

---

## Bots (Jade / Pixieglow)

Both bots are thin WebSocket clients of Emerald. They have no Sapphire or LLM logic. All behavior decisions (typo, burst, voice chance, sleep) are made by Emerald.

### Jade (Discord)

- TypeScript + Eris library
- Connects to Emerald via WebSocket and forwards Discord messages as `MessageEvent`s
- `-debug` toggle enables per-message debug mode
- Executes Emerald commands: `respond`, `typing`, `set_presence`, reaction plans, burst fragments
- Generates Piper TTS voice messages when Emerald sets `voice: true`
- In debug mode, appends `-# ` formatted stat lines (tokens, timing, emotion, behavior config)
- Config: `discord_token`, `emerald_host`, `emerald_port`, TTS paths
- Key file: `src/bot.ts`

### Pixieglow (Matrix)

- TypeScript + Bun + Matrix client-server API
- Sync loop polls Matrix API for new messages, forwards to Emerald
- Executes Emerald commands: `respond`, `typing`, `set_presence`
- Config: `matrix.homeserver_url`, `matrix.access_token`, `emerald_host`, `emerald_port`
- Key file: `src/bot/matrix-bot.ts`

---

## WebSocket Protocol

Emerald communicates with bots via JSON over WebSocket.

### Events (Bot &rarr; Emerald)

| Event | Fields | Description |
|---|---|---|
| `message` | `id`, `client`, `channel`, `user`, `text`, `timestamp`, `isDM`, `mentions?`, `debug?` | User message from platform |
| `ready` | `client`, `userId`, `username` | Bot connection handshake |
| `bot_message` | `client`, `channel`, `text`, `timestamp` | Bot's own messages (for context) |
| `presence` | `client`, `status` | Bot status update |

### Commands (Emerald &rarr; Bot)

| Command | Fields | Description |
|---|---|---|
| `respond` | `id`, `channel`, `text`, `responseText`, `delay`, `replyTo?`, `replyStyle`, `hesitationWord?`, `burstPlan?`, `react?`, `voice?`, `sessionId?`, `debugStats?` | Response to post |
| `typing` | `id`, `channel`, `duration` | Show typing indicator |
| `set_presence` | `id`, `status`, `text?`, `activityType?` | Update bot presence |
| `forgot` | `id`, `channel` | Bot forgot conversation |
| `spontaneous` | `id`, `channel`, `sessionId` | Initiate spontaneous message |

### WebSocket Messages

| Event | Payload |
|---|---|
| `"command"` | `command: OutCommand` |
| `"ack"` | `ackId: string` |

### DebugStats

```typescript
{
  promptTokens: number;
  completionTokens: number;
  timeMs: number;
  tokensPerSecond: number;
  emotionStateValence: number;
  emotionStateArousal: number;
  classificationLabel: string;
  classificationConfidence: number;
  messageValence: number;
  messageArousal: number;
  triggerReason: string;
  delay: number;
  usedRuby: boolean;
  inactivityMs: number;
  behavior?: BehaviorDebug;
}

BehaviorDebug {
  typoChance, typoApplied,
  swapChance, swapApplied,
  burstChance, burstApplied,
  hesitationChance, hesitationApplied,
  voiceChance, voiceApplied,
  forgetChance,
  sleepMode: string | null,
  fatigueMultiplier: number,
}
```

---

## Debug Mode

Toggle by typing `-debug` in any channel:

1. **Bot level**: Message with `-debug` prefix toggles debug for that channel, sets `debug: true` on subsequent `MessageEvent`s
2. **Emerald**: Passes `debug: true` to Sapphire's `/v1/respond`
3. **Sapphire**: Returns debug fields in the response
4. **Emerald**: Maps snake_case &rarr; camelCase into `DebugStats` + `BehaviorDebug`, included in `RespondCommand`
5. **Bot**: Appends `-# ` formatted lines to the response

Example debug output:

```
Hey, how are you?
-# 🚀 12 tok · 3200ms · 3.8 tok/s · 450 prompt
-# 😊 valence=0.014 · arousal=-0.040
-# 📊 state v=0.023 · a=0.012
-# 🏷️ FUTILE (42.1%)
-# ⚙️ typo=6% · swap=4% · burst=15% · hesitate=15% · voice=12% ✨ · forget=3% · fatigue=1.0x
```

---

## Configuration

### Emerald (`config.yml`)

All behavior configuration centralized in Emerald:

```yaml
port: 3126
sapphire_host: "127.0.0.1"
sapphire_port: 3123
names: ["Luna", "Pixie"]
random_chance: 0.015
cooldown_seconds: 8
burst_chance: 0.15
# See config.example.yml for full details
```

### Sapphire (`config.yml` or env vars)

Sapphire loads from `config.yml` in the repo root. Shell env vars with the `SAPPHIRE_` prefix override YAML keys at startup.

| Key | Default | Description |
|---|---|---|
| `port` | `3123` | Server port |
| `krystal_generic_url` | `http://127.0.0.1:3124` | Backend for FUTILE |
| `krystal_semantic_url` | `http://127.0.0.1:3124` | Backend for INTERESSANT |
| `bot_name` | `"Luna"` | Bot persona name |
| `system_prompt` | `"Your name is Luna..."` | Full system prompt |
| `few_shot_enabled` | `true` | Enable/disable few-shot priming |
| `few_shot_examples_path` | `./few_shot_examples.yml` | Path to examples file |
| `emotion_decay` | `0.85` | EMA decay factor |
| `emotion_deadzone` | `0.005` | Noise threshold |
| `mirostat_enabled` | `true` | Enable mirostat sampling |
| `llm_max_retries` | `2` | Retries on degenerate output |

### Bots

Both bots only need platform auth keys, Emerald connection details, and TTS paths (Jade only). No behavior parameters.

**Jade (Discord):**
```yaml
discord_token: "your_token"
emerald_host: "localhost"
emerald_port: 3126
tts_model_path: "./tts-engine/en_GB-southern_english_female-low.onnx"
tts_binary_path: "bin/piper/piper"
```

**Pixieglow (Matrix):**
```yaml
matrix:
  homeserver_url: "https://matrix.example.com"
  access_token: "your_token"
emerald_host: "localhost"
emerald_port: 3126
```

---

## Dataset

[**Discord-Dialogues**](https://huggingface.co/datasets/mookiezi/Discord-Dialogues) -- 7.3M exchanges, 17M turns, 140M words. Real Discord conversations spring-summer 2025, filtered PII/ToS/bots/commands. Apache 2.0.

| Metric | Value |
|---|---|
| Samples | 7 303 464 |
| Total turns | 16 881 010 |
| Total words | 139 922 950 |
| Average tokens | 32.8 |

Model: `Luna-Protocol-1.5B-Fine-Tuned-Qwen2.5-200k-instruct` -- fine-tuned on a 200k sample subset (Q4_K_M, 941 MB).

---

## Discord Developer Portal

- **Message Content Intent** (Bot tab)
- Scope `bot` + permissions: `Send Messages`, `Read Message History`, `Add Reactions`
- Gateway intents: `guilds`, `guildMessages`, `guildMessageReactions`, `messageContent`, `directMessages`
