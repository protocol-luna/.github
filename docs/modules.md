# Module Details

Deep-dive into each module: architecture, features, configuration, and developer notes.

## Krystal -- LLM Inference Server

**Language:** C++ (llama.cpp) · **Port:** 3124 · **Runtime:** PM2

Krystal wraps llama.cpp's `llama-server` to serve GGUF models via an OpenAI-compatible HTTP API. It's the lowest layer of the stack -- the actual neural network inference.

### Architecture

Sapphire routes FUTILE messages to `KRYSTAL_GENERIC_URL` (`:3124` by default) and INTERESSANT to `KRYSTAL_SEMANTIC_URL` (`:3125` by default). In our deployment both point to the same Krystal instance via `KRYSTAL_SEMANTIC_URL` env var override.

### Features

- **OpenAI-compatible API**: `/v1/chat/completions` endpoint works with standard OpenAI client libraries
- **Slot-based concurrency**: llama.cpp slot system allows multiple simultaneous inference contexts
- **Prompt caching**: `cache_prompt: true` reuses cached computation across requests
- **Streaming SSE**: Native token-by-token streaming support
- **Sampling controls**: Full llama.cpp sampler suite (temperature, top_k, top_p, min_p, mirostat, repeat_penalty, etc.)

### Configuration

Krystal is started via `start.sh` which passes parameters to `llama-server`:

```bash
./llama-server \
  --model models/Luna-Protocol-1.5B-Fine-Tuned-Qwen2.5-200k-instruct.Q4_K_M.gguf \
  --host 127.0.0.1 --port 3124 \
  --ctx-size 4096 --n-gpu-layers 0 --threads 6 \
  --parallel 1
```

| Parameter | Value | Notes |
|---|---|---|
| Model | Qwen2.5 1.5B Q4_K_M | 941 MB, fine-tuned on Discord dialogues |
| Context | 4096 tokens | Sufficient for ~20 message pairs + prompt |
| GPU layers | 0 | CPU-only inference |
| Threads | 6 | One per CPU core available |
| Parallel slots | 1 | Single concurrent request |

### Developer Notes

- No Sapphire-specific changes needed -- standard llama.cpp server with default endpoint
- To add a second backend, set `KRYSTAL_SEMANTIC_URL` to a different port and run another instance
- Model swapping requires PM2 restart: `pm2 restart krystal-small`
- Disk space for models: ~1 GB per Q4_K_M 1.5B model

### Links

- [GitHub: protocol-luna/krystal](https://github.com/protocol-luna/krystal)
- [llama.cpp](https://github.com/ggml-ai/llama.cpp)

## Sapphire -- LLM Gateway

**Language:** Python (FastAPI) · **Port:** 3123 · **Runtime:** PM2 (venv/python)

Sapphire sits between Emerald (the brain) and Krystal (llama.cpp). It's called exclusively by Emerald and handles all LLM-related logic: classification, session management, few-shot injection, emotion scoring, degenerate detection, and prompt construction.

### Architecture

Sapphire is a stateless HTTP server (Python/FastAPI) with in-memory session state. Each request to `/v1/respond` goes through: embedding → classification → session lookup → few-shot injection → prompt construction → Krystal call → degeneracy check → response.

### Features

#### Embedding Classification (K-Means Multicentroid)

Uses `fastembed` with `BAAI/bge-small-en-v1.5` (384-dim embeddings). At startup, **k-means clustering** (k=10, seed=42) computes 10 centroids per class from YAML example files:

- **FUTILE/INTERESSANT**: ~1100 examples from `examples.yml` → 10 centroids per class
- **Emotion axes**: 83 positive, 79 negative, 85 high-arousal, 116 low-arousal from `examples_emotion.yml` → 10 centroids per pole

Classification uses **maximum cosine similarity** across all 10 centroids per class, not a single centroid. The fixed seed (42) ensures reproducible centroids across rebuilds. Achieves **100% accuracy on the 396-example test suite**.

#### Session Management

Each conversation (keyed by `session_id`) stores up to 20 message pairs. Sessions expire after 600s of inactivity. The session store maps sessions to llama.cpp slots for prompt caching.

#### Few-Shot Injection

5 example exchanges from `few_shot_examples.yml` are injected between the system prompt and conversation history. The user's display name is prepended to each user example. Configurable via `SAPPHIRE_FEW_SHOT_ENABLED`.

#### Emotion-Aware Sampling

Per-message valence/arousal scores update a running emotional state (EMA decay 0.85, deadzone 0.005). Sampling parameters sent to Krystal are adjusted:

```python
temperature    = clamp(0.7 + arousal * 0.3,   0.4, 1.0)
repeat_penalty = clamp(1.15 - valence * 0.1,  1.0, 1.3)
mirostat_ent   = clamp(6.0  + arousal * 2.0,  3.0, 8.0)
```

#### Degenerate Detection

Regex checks for empty, too-short, or character-repetitive outputs. In non-streaming mode, Sapphire retries up to 2 times. In streaming mode, the full response is buffered before checking -- if degenerate, the stream ends silently with no metadata.

#### Streaming SSE

When `stream: true` is set, Sapphire returns a Server-Sent Events stream:

```
data: Hello
data:  world
data: !
data: {"text":"Hello world!","label":"FUTILE","valence":0.1,"arousal":-0.05,"time_ms":3200,"tokens_per_second":3.8}
data: [DONE]
```

Each content token is a `data: <text>` line. The final JSON metadata includes the full text, classification, emotion scores, and debug fields (if `debug: true`). If the output is degenerate, the stream stops at `[DONE]` without metadata.

### Configuration

All configuration is via environment variables:

| Variable | Default | Description |
|---|---|---|
| `SAPPHIRE_PORT` | 3123 | HTTP server port |
| `KRYSTAL_GENERIC_URL` | `http://127.0.0.1:3124` | FUTILE backend |
| `KRYSTAL_SEMANTIC_URL` | `http://127.0.0.1:3125` | INTERESSANT backend |
| `SAPPHIRE_BOT_NAME` | `"Luna"` | Bot persona name |
| `SAPPHIRE_SYSTEM_PROMPT` | `"Your name is Luna..."` | Full system prompt |
| `SAPPHIRE_EMOTION_DEADZONE` | 0.005 | Noise threshold for emotion deltas |
| `SAPPHIRE_EMOTION_DECAY` | 0.85 | EMA decay for emotion state |
| `SAPPHIRE_FEW_SHOT_ENABLED` | true | Enable few-shot injection |
| `SAPPHIRE_LLM_MAX_HISTORY` | 20 | Max message pairs in context |
| `SAPPHIRE_LLM_MAX_RETRIES` | 2 | Degenerate output retries |
| `SAPPHIRE_MIROSTAT_ENABLED` | true | Mirostat sampling |

### Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/v1/respond` | POST | Main: classify + generate response (streaming or non) |
| `/v1/reset` | POST | Reset session(s) |
| `/classify` | POST | Classify text without generation |
| `/emotion/{key}` | GET | Get emotional state for a conversation |
| `/health` | GET | Health check with system status |

### Developer Notes

- Runs under `./venv/bin/python` (not system python)
- Model centroids are computed at startup (~10s for embedding + classification)
- Debug fields use `snake_case` in Sapphire JSON but are mapped to `camelCase` in Emerald's TypeScript types
- The system prompt must explicitly forbid echoing (the 1.5B model tends to repeat user input otherwise)

### Links

- [GitHub: protocol-luna/sapphire](https://github.com/protocol-luna/sapphire)

## Emerald -- Brain Service

**Language:** TypeScript · **Port:** 3126 (WebSocket) · **Runtime:** PM2 (esbuild → self-cli.cjs)

Emerald is the central decision engine. Bots connect via WebSocket and Emerald orchestrates the entire response pipeline: trigger evaluation, behavior decisions, Sapphire streaming, response text processing, and command dispatch.

### Architecture

Emerald runs a WebSocket server on port 3126. Bots (Jade, Pixieglow) connect, register with a `ready` event, and forward `message` events. Emerald evaluates triggers, calls Sapphire, processes the response, and sends commands back.

### Features

#### Streaming + First-Token Typing

Emerald calls Sapphire with `stream: true` and consumes the SSE response token by token. **The first token immediately triggers a `TypingCommand` to the bot** -- the user sees "typing..." before the model has finished generating. This replaces the old fixed-delay approach where typing started after a computed wait.

The streaming response is buffered in Emerald; when the full text is received, degeneracy is checked (Sapphire-side), and the `RespondCommand` is sent. If degenerate, typing expires naturally and no message is posted.

#### Centralized Behavior Config

All behavioral parameters live in a single `config.yml`: typo chance/layout, letter swap chance, burst chance/delays, hesitation chance/word list, sleep schedules, topic fatigue thresholds, voice message chance, forget chance, and all trigger delays. Bots have zero behavior configuration.

#### Voice Message Decision

Emerald decides whether a response should be a voice message based on `voice_message_chance` (default 12%). If the roll succeeds, the `RespondCommand` includes `voice: true`. The bot (Jade) then generates Piper TTS audio and uploads it. The chance is evaluated per-message, not per-session.

#### Mention Stripping

Before sending text to Sapphire, Emerald strips bot mentions (`@Kalupso`, `<@userId>`) from the message content. This prevents the model from seeing its own name in the input, which reduces echo behavior.

#### Debug Mode

When `debug: true` is set on a `MessageEvent`, Emerald passes it to Sapphire which returns token counts, timing, emotion state, and classification confidence. Emerald maps snake_case → camelCase and includes a `BehaviorDebug` snapshot showing:
- All configured behavior chances (typo, swap, burst, hesitate, voice, forget)
- Which behaviors were triggered for this specific message (`voiceApplied`, `typoApplied`, etc.)
- Current sleep mode and fatigue multiplier

#### Behavior Processing Pipeline

1. **Pre-Sapphire**: trigger evaluation, ignore/forget checks, burst/hesitation/voice decisions, react plan, delay computation
2. **Sapphire call**: streaming request with stripped text
3. **Post-Sapphire**: typo injection, letter-swap, hesitation marker prepending, debug stats construction
4. **Command dispatch**: typing command (via callback on first token), then respond command

### Protocol

Bots communicate with Emerald via WebSocket JSON messages:

```json
// Bot → Emerald (event)
{"event": "in", "payload": {"type": "message", "id": "...", "client": "jade", ...}}

// Emerald → Bot (command)
{"event": "command", "command": {"type": "respond", "id": "...", "responseText": "...", ...}}
```

#### Events (Bot → Emerald)

| Type | Key fields | Purpose |
|---|---|---|
| `ready` | client, userId, username | Handshake -- register bot identity |
| `message` | id, channel, user, text, timestamp, isDM, mentions?, debug? | Forward user message |
| `bot_message` | client, channel, text, timestamp | Bot's own output for context tracking |
| `presence` | client, status | Bot status update |

#### Commands (Emerald → Bot)

| Type | Key fields | Purpose |
|---|---|---|
| `respond` | id, channel, responseText, delay, replyTo, replyStyle, hesitationWord?, burstPlan?, react?, voice?, debugStats? | Send response text (optionally as voice) |
| `typing` | id, channel, duration | Show typing indicator |
| `set_presence` | id, status, text?, activityType? | Update bot presence |
| `forgot` | id, channel | Bot forgot the conversation |
| `spontaneous` | id, channel, sessionId | Initiate spontaneous message |

### File Structure

```
emerald/
  src/
    server.ts            WebSocket server, connection management
    brain.ts             Decision engine, streaming Sapphire calls
    sapphire-client.ts   Sapphire HTTP client (ask/askStream)
    protocol.ts          TypeScript type definitions
    config.ts            YAML config loader
    behavior/
      burst.ts           Burst message logic
      mannerisms.ts      Hesitation, reactions, delays
      sleep.ts           Sleep schedule evaluation
      typo.ts            Typo/letter-swap application
    state/
      state.ts           Activity tracking, session limits
      trigger.ts         Trigger evaluation (mentions, names, etc.)
      topic-fatigue.ts   Topic fatigue tracking
  config.example.yml     All behavior defaults documented
  self-cli.cjs           Compiled PM2 entry point
```

### Configuration

Key parameters in `config.yml`:

```yaml
port: 3126
sapphire_host: "127.0.0.1"
sapphire_port: 3123
sapphire_bot_username: "User"
names: ["Luna", "Pixie"]
keywords: ["hello", "hi", "hey", ...]
random_chance: 0.015
burst_chance: 0.15
hesitation_chance: 0.15
typo_chance: 0.06
letter_swap_chance: 0.04
voice_message_chance: 0.12
forget_chance: 0.03
# + sleep schedules, topic fatigue, reply styles, concentrations...
```

### Developer Notes

- Built with esbuild → `self-cli.cjs`. Always run `npm run build` before `pm2 restart emerald`
- WebSocket server uses the `ws` library
- Bot mention stripping uses regex: `<@!?userId>` and `@username`
- The `sendCommand` callback in `handleEvent` enables early command dispatch (typing before respond)
- Debug stats mapping: `debug_prompt_tokens` → `debugPromptTokens` (snake_case in Sapphire JSON → camelCase in TypeScript)

### Links

- [GitHub: protocol-luna/emerald](https://github.com/protocol-luna/emerald)

## Jade -- Discord Adapter

**Language:** TypeScript · **Library:** Eris · **Runtime:** PM2 (esbuild → self-cli.cjs)

Jade connects to Discord and acts as a transparent WebSocket proxy to Emerald. It has no LLM logic, no behavior decisions, and no Sapphire calls. All intelligence is in Emerald.

### Architecture

Jade runs two connections:
1. **Discord gateway** (Eris client): receives messages, sends responses, manages typing indicators
2. **Emerald WebSocket**: forwards messages as `MessageEvent`s, receives `OutCommand`s

### Features

#### Message Forwarding

Every Discord message (except the bot's own) is forwarded to Emerald as a `MessageEvent`. The bot's own messages are sent as `BotMessageEvent` to maintain conversational context.

#### Command Execution

Jade handles the full command set from Emerald:
- **respond**: Posts `responseText` with optional reply styling, hesitation markers, burst fragments, reactions, and debug stats
- **typing**: Shows Discord typing indicator for the specified duration
- **set_presence**: Updates the bot's Discord status and activity text
- **forgot**: Posts a "what were we talking about?" type message
- **spontaneous**: Reserved for future use (initiating conversations)

#### TTS Voice Messages

When Emerald sends `voice: true` on a respond command, Jade generates audio using Piper TTS (local binary + ONNX model), converts to OGG, uploads to Discord's CDN, and posts as a voice message. The decision to send voice is made by Emerald (`voice_message_chance`), not by Jade.

#### Debug Mode

Typing `-debug` in a channel toggles debug mode for that channel. Subsequent responses include appended `-# ` lines showing:

```
-# 🚀 12 tok · 3200ms · 3.8 tok/s · 450 prompt
-# 😊 valence=0.014 · arousal=-0.040
-# 📊 state v=0.023 · a=0.012
-# 🏷️ FUTILE (42.1%)
-# ⚙️ typo=6% · swap=4% · burst=15% · hesitate=15% · voice=12% ✨ · forget=3% · fatigue=1.0x
```

#### Reaction Handling

The bot auto-reacts to certain emoji combinations on its own messages (✅ for ❌, ▶️, 🗑️ reactions) to simulate acknowledgment.

### Configuration

Config (`config.yml`) is minimal -- no behavior parameters at all:

```yaml
discord_token: "your_bot_token"
emerald_host: "127.0.0.1"
emerald_port: 3126
tts_model_path: "./tts-engine/en_GB-southern_english_female-low.onnx"
tts_binary_path: "bin/piper/piper"
ffmpeg_path: "bin/ffmpeg/ffmpeg"
ffprobe_path: "bin/ffmpeg/ffprobe"
```

### Developer Notes

- Built with esbuild → `self-cli.cjs`. Run `npm run build` before `pm2 restart jade`
- Discord intents required: guilds, guildMessages, guildMessageReactions, messageContent, directMessages
- TTS engine: Piper with southern English female voice (ONNX model, ~20 MB)
- Voice message flow: sanitize text → Piper synthesize WAV → ffmpeg convert to OGG → Discord CDN upload → post voice message

### Links

- [GitHub: protocol-luna/jade](https://github.com/protocol-luna/jade)
- [Eris (Discord library)](https://github.com/abalabahaha/eris)
- [Piper TTS](https://github.com/rhasspy/piper)

## Pixieglow -- Matrix Adapter

**Language:** TypeScript · **Runtime:** Bun · **Protocol:** Matrix Client-Server API

Pixieglow is the Matrix equivalent of Jade. It connects to a Matrix homeserver and forwards room messages to Emerald via WebSocket. Identical architecture, different platform.

### Architecture

Pixieglow uses a polling sync loop against the Matrix client-server API. It reads room messages, filters out the bot's own events, and forwards user messages to Emerald. Responses from Emerald are posted to the room.

### Features

- **Sync loop**: Polls `/_matrix/client/v3/sync` with a `since` token for incremental updates
- **Multi-room**: Responds in all rooms the bot is a member of
- **Emerald commands**: Supports `respond`, `typing` (Matrix typing indicator), `set_presence`
- **Minimal config**: Just homeserver URL, access token, and Emerald connection details

### Configuration

```yaml
matrix:
  homeserver_url: "https://matrix.fox3000foxy.com"
  access_token: "your_matrix_access_token"
emerald_host: "127.0.0.1"
emerald_port: 3126
```

### Developer Notes

- Runs on Bun (not Node.js)
- No TTS support (Matrix doesn't have Discord-style voice messages)
- No debug mode formatting (no `-# ` lines -- Matrix doesn't support Discord-style markup)
- Tested with [tuwunel](https://github.com/tuwunel/tuwunel) homeserver
- Matrix user: `@pixieglow:protocol-luna.github.io` (delegated via well-known)

### Links

- [GitHub: protocol-luna/pixieglow](https://github.com/protocol-luna/pixieglow)
- [Matrix Client-Server API](https://spec.matrix.org/v1.13/client-server-api/)

## Ruby -- Markov Chain Service

**Language:** TypeScript · **Port:** 3127 · **Runtime:** PM2 (esbuild → self-cli.cjs)

Ruby builds and serves an order-2 Markov chain from all messages flowing through Emerald. It generates ambient messages and spontaneous replies without LLM latency. It's the only service in the stack that *learns from conversation data* in real time.

### Architecture

Ruby is a simple HTTP server. Emerald sends every incoming message to Ruby's `/train` endpoint (fire-and-forget) and calls `/generate` for random triggers. Ruby stores its chain in a persistent SQLite database via `better-sqlite3` (native binding), with `TRANSACTIONS` and `STARTERS` tables indexed by channel. WAL mode ensures immediate durability without periodic snapshots.

### Features

- **Order-2 Markov chain**: Each state is a 2-word prefix mapping to possible next words with weighted random selection. Lower order produces more varied output while remaining coherent.
- **Real-time training**: Every message that flows through Emerald is forwarded to `/train`. The chain grows continuously as the bot converses.
- **Seeded generation**: Emerald can pass a seed phrase and max length. Ruby walks the chain from any matching 2-word starter.
- **Channel-aware**: Training and generation can be scoped to specific channels via `channel_id`.
- **Zero LLM cost**: Ruby responses use no GPU, no VRAM, no Krystal calls. Ambient messages cost ~1ms of CPU time.

### Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/train` | POST | Train on a message. Body: `{"text": "hello there", "channel_id": "..."}` |
| `/train-batch` | POST | Train multiple messages atomically in a SQL transaction |
| `/generate` | POST | Generate text. Body: `{"seed": "how are you", "max_length": 30, "channel_id": "..."}` |
| `/channels` | GET | List all known channel IDs |
| `/stats` | GET | Returns transition count, starter count, and database size |
| `/health` | GET | Health check |

### Configuration

```yaml
port: 3127
host: "127.0.0.1"
order: 2
max_length: 30
skip_dm: true
db_path: "chain.db"
```

### How the Markov Chain Works

Ruby tokenizes messages by stripping URLs, mentions, and punctuation -- keeping only word characters and apostrophes. Each message produces a sequence of tokens. Every sliding window of 2 words records the following word with a frequency count. Generation picks a random 2-word starter and iteratively selects the next word via weighted random sampling:

```
Input: "how are you doing today"
Tokens: how, are, you, doing, today
Prefix→suffix:
  "how are" → you
  "are you" → doing
  "you doing" → today

Generation: pick starter → weighted random → slide window → repeat
```

With order 2, Ruby produces varied but coherent sentences at minimal memory cost.

### Emerald Integration

Emerald includes Ruby in its trigger pipeline. When a message is triggered by a `random` or `spontaneous` reason, Emerald calls `ruby.generate()` instead of Sapphire. Every human message is also forwarded to `/train` (fire-and-forget). This means ambient messages cost virtually nothing. The integration is transparent to the bot -- it receives a `RespondCommand` either way.

### Developer Notes

- Built with esbuild → `self-cli.cjs`. Run `npm run build` before `pm2 restart ruby`
- SQLite uses `better-sqlite3` (native C binding, not WASM) -- writes are immediately durable, no save interval needed
- WAL mode enabled; VACUUM runs automatically every 500K transitions
- Seed lookup: multi-word seeds are matched with LIKE + JS startsWith filter (avoids C string truncation)
- Pre-trained chain (order 2) available on [HuggingFace](https://huggingface.co/fox3000foxy/ruby-chain)

### Links

- [GitHub: protocol-luna/ruby](https://github.com/protocol-luna/ruby)
