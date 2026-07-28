# Getting Started

From zero to running bot in 20 minutes. All local, no cloud APIs.

## Quick Start (20 min)

Get a fully working bot in 8 commands:

```bash
# 1. Start the LLM server (Krystal)
git clone https://github.com/protocol-luna/krystal
cd krystal
./start.sh   # Luna 1.5B on :3124

# 2. Start the LLM gateway (Sapphire)
git clone https://github.com/protocol-luna/sapphire
cd sapphire
pip install -r requirements.txt
python server.py &   # listens on :3123

# 3. Start the brain (Emerald)
git clone https://github.com/protocol-luna/emerald
cd emerald
npm install
cp config.example.yml config.yml
npm start &   # WebSocket on :3126

# 4. Start a bot (example: Jade for Discord)
git clone https://github.com/protocol-luna/jade
cd jade
cp config.example.yml config.yml
# edit config.yml: set discord_token, emerald_host=localhost, emerald_port=3126
npm start
```

## Architecture Overview

The Luna Protocol stack has four layers:

```
Platform Layer    ┌──────────────┐  ┌───────────────┐
(Discord/Matrix)  │   Discord    │  │    Matrix     │
                  └──────┬───────┘  └───────┬───────┘
                         │                  │
Adapter Layer     ┌──────▼───────┐  ┌───────▼───────┐
(Jade/Pixieglow)  │    Jade      │  │   Pixieglow   │
                  │ (WebSocket)  │  │  (WebSocket)  │
                  └──────┬───────┘  └───────┬───────┘
                         │                  │
                         └────────┬─────────┘
                                  │ WebSocket :3126
Brain Layer       ┌────────────────▼───────────────┐
(Emerald)         │      Emerald (Brain)           │
                  │  behavior + Sapphire + Ruby    │
                  └──────┬────────────┬────────────┘
                         │ HTTP :3123 │ HTTP :3127
                  ┌──────▼──────┐    ┌▼────────────┐
                  │  Sapphire   │    │   Ruby      │
                  │  (Gateway)  │    │  (Markov)   │
                  └──────┬──────┘    └─────────────┘
                         │ HTTP :3124 / :3125
                  ┌──────▼────────────────────┐
                  │  Krystal (llama.cpp)      │
                  │  single or dual instance  │
                  └───────────────────────────┘
```

## Prerequisites

| Component | Requirement |
|---|---|
| Krystal | Linux (llama-server), ~1 GB disk for model |
| Sapphire | Python 3.10+, ~200 MB for fastembed cache |
| Emerald | Node.js 18+ |
| Jade (Discord) | Node.js 18+, Discord bot token |
| Pixieglow (Matrix) | Bun, Matrix access token + homeserver URL |

## Step 1: Krystal (LLM Server)

Krystal wraps `llama-server` from llama.cpp. You can run two instances -- one for FUTILE messages on port 3124 (smaller model, faster) and one for INTERESSANT messages on port 3125 (larger model, smarter), or point both Sapphire routes to the same instance via environment variables.

```bash
git clone https://github.com/protocol-luna/krystal
cd krystal
./start.sh   # starts on :3124 with Luna 1.5B Q4_K_M
```

**Download the model:**

```bash
huggingface-cli download \
  fox3000foxy/Luna-Protocol-1.5B-Discord-Dialogues-200k-instruct \
  --local-dir models/
```

**Environment variables:**

| Variable | Description | Default |
|---|---|---|
| `KRYSTAL_MODEL_PATH` | Path to GGUF file | ./models/Luna-1.5B...gguf |
| `KRYSTAL_PORT` | Server port | 3124 |
| `KRYSTAL_N_THREADS` | Thread count | 6 |
| `KRYSTAL_N_CTX` | Context size | 4096 |

## Step 2: Sapphire (Gateway)

Sapphire sits between Emerald and Krystal. It classifies messages, manages sessions, injects few-shot examples, and detects degenerate responses.

```bash
git clone https://github.com/protocol-luna/sapphire
cd sapphire
pip install -r requirements.txt
python server.py
# listens on 127.0.0.1:3123
```

**How it works:**

- Loads `BAAI/bge-small-en-v1.5` (fastembed, auto-downloaded on first run, ~100 MB)
- Computes k-means multicentroids (k=10, seed=42) from `examples.yml` (~1100 curated examples)
- Each message is embedded and compared to centroids via max cosine similarity across 10 per class
- Scores emotion (valence/arousal) on [-1, 1] scale via multicentroid comparison
- Manages per-channel conversation sessions with configurable history length
- Calls Krystal with emotion-aware sampling parameters

**Key endpoints (Emerald uses /v1/respond):**

| Endpoint | Method | Description |
|---|---|---|
| `/v1/respond` | POST | Classify + session + few-shot + emotion + call Krystal. Returns text and optional debug stats. |
| `/v1/chat/completions` | POST | OpenAI-compatible chat completions (no sessions, no few-shot). For external tools. |
| `/classify` | POST | Classify only, returns label + similarity scores + emotion scores. |
| `/health` | GET | Server status + configuration. |

**Environment variables:**

| Variable | Default | Description |
|---|---|---|
| `SAPPHIRE_PORT` | `3123` | Server port |
| `KRYSTAL_GENERIC_URL` | `http://127.0.0.1:3124` | Backend for FUTILE messages |
| `KRYSTAL_SEMANTIC_URL` | `http://127.0.0.1:3125` | Backend for INTERESSANT messages |
| `SAPPHIRE_EXAMPLES` | `./examples.yml` | Path to example list |
| `SAPPHIRE_SYSTEM_PROMPT` | `"Your name is Luna..."` | System prompt for all conversations |

## Step 3: Emerald (Brain)

Emerald is the central WebSocket server. Bots connect to it, and it orchestrates all LLM and behavior logic.

```bash
git clone https://github.com/protocol-luna/emerald
cd emerald
npm install
cp config.example.yml config.yml
# edit config.yml if needed (sapphire_host, behavior settings)
npm start
# WebSocket server on 127.0.0.1:3126
```

**What Emerald does:**

- Accepts WebSocket connections from bots on port 3126
- Evaluates behavior rules: burst cooldown, sleep schedule, typo/swap, mannerisms, ignore prefixes
- Calls Sapphire's `/v1/respond` for non-ignored messages
- Applies typo/swap behavior to the response
- Sends `RespondCommand` back to the bot with `responseText` and optional `debugStats`
- Sends `TypingCommand` for typing indicators

### Configuration (config.yml)

```yaml
port: 3126
sapphire_host: "localhost"
sapphire_port: 3123
names: ["Luna", "Pixie"]
keywords: ["hello", "hi", "hey", ...]
random_chance: 0.015
cooldown_seconds: 8
burst_chance: 0.15
hesitation_chance: 0.15
typo_chance: 0.06
letter_swap_chance: 0.04
voice_message_chance: 0.12
forget_chance: 0.03
# See config.example.yml for full details
```

## Step 4: Bot (Jade / Pixieglow)

Both bots are thin WebSocket clients of Emerald. They have no Sapphire or LLM logic.

### Jade (Discord)

```bash
git clone https://github.com/protocol-luna/jade
cd jade
cp config.example.yml config.yml
# edit config.yml: set discord_token, emerald_host=localhost, emerald_port=3126
npm install
npm start
```

**Config (config.yml):**

```yaml
discord_token: "your_discord_bot_token"
emerald_host: "localhost"
emerald_port: 3126
# Optional TTS
tts:
  model_path: "/path/to/model.onnx"
  config_path: "/path/to/model.json"
  output_dir: "/tmp/tts"
  voice_chance: 0.05
```

Generate an OAuth2 URL from the [Discord Developer Portal](https://discord.com/developers/applications): scope `bot`, permissions `Send Messages`, `Read Message History`, `Add Reactions`, intents `guilds`, `guildMessages`, `guildMessageReactions`, `messageContent`, `directMessages`.

### Pixieglow (Matrix)

```bash
git clone https://github.com/protocol-luna/pixieglow
cd pixieglow
cp config.example.yml config.yml
# edit config.yml: set matrix access token, emerald_host=localhost, emerald_port=3126
bun install
bun start
```

**Config (config.yml):**

```yaml
matrix:
  homeserver_url: "https://matrix.example.com"
  access_token: "your_matrix_access_token"
emerald_host: "localhost"
emerald_port: 3126
```

The bot auto-joins rooms it's invited to. Requires a Matrix homeserver (tested with [tuwunel](https://github.com/tuwunel/tuwunel)). Invite `@pixieglow:protocol-luna.github.io` on the Luna server to test.

## Step 5: Ruby -- Markov Chain (Optional)

Ruby builds an order-2 Markov chain from all messages flowing through Emerald. It generates ambient messages and spontaneous replies without LLM latency.

```bash
git clone https://github.com/protocol-luna/ruby
cd ruby
npm install
npm run build
npm start   # HTTP server on :3127
```

Enable Ruby in Emerald's config.yml:

```yaml
ruby_enabled: true
ruby_host: "localhost"
ruby_port: 3127
ruby_reasons: ["random", "spontaneous"]
```

Optionally download a pre-trained chain (trained on real Discord messages):

```bash
npm run download-chain
```

Once enabled, Emerald automatically trains Ruby on every human message and uses it for low-priority triggers (random and ambient messages), bypassing the LLM.

## Testing Your Setup

Verify each layer independently:

```bash
# 1. Krystal is running
curl http://127.0.0.1:3124/health

# 2. Sapphire is running
curl http://127.0.0.1:3123/health

# 3. Classification works
curl -X POST http://127.0.0.1:3123/classify \
  -H 'Content-Type: application/json' \
  -d '{"text":"hello"}'
# Expected: {"label":"FUTILE",...}

curl -X POST http://127.0.0.1:3123/classify \
  -H 'Content-Type: application/json' \
  -d '{"text":"how does quantum computing work"}'
# Expected: {"label":"INTERESSANT",...}

# 4. Full /v1/respond pipeline
curl -X POST http://127.0.0.1:3123/v1/respond \
  -H 'Content-Type: application/json' \
  -d '{"username":"test","text":"hello","session_id":"test","behavior":{},"messages":[{"role":"user","content":"hello"}]}'
# Expected: {"text":"...","debug":false}
```

Run the classifier test suite:

```bash
cd sapphire
python test_classifier.py
```

## Debug Mode

Any message can be processed in debug mode to see token counts, timing, and emotion state.

**On Discord (Jade):** prefix your message with `-debug`

```
-debug hello, how are you?
# Bot responds:
# i'm doing well, thanks for asking!
# - 12 tokens | 450 in | 3.2s | 3.75 t/s | emo: +0.6v/0.3a | conf: 0.85
```

Debug output includes: prompt tokens, completion tokens, time in seconds, tokens/sec, emotional valence/arousal, and classification confidence.

## Best Practices

### Production Deployment

- **Use PM2** for auto-restart: all four services should be managed by PM2
- **Monitor disk space**: GGUF model is ~1 GB; logs can grow
- **Use a reverse proxy** (nginx, cloudflared) if exposing endpoints outside localhost
- **Order matters**: start Krystal → Sapphire → Emerald → bots

### Hardware Requirements

| Scenario | RAM | Disk | Notes |
|---|---|---|---|
| Minimum | 3 GB | 2 GB | Luna 1.5B Q4_K_M, CPU only |
| Recommended | 4-8 GB | 4 GB | Room for context window + logs |

### Tuning Examples

If messages are misclassified, edit `sapphire/examples.yml`. The k-means multicentroid (k=10, seed=42) automatically captures sub-types across 10 centroids per class:

- Add more examples of the misbehaving type
- Use weighted items: `{text: "example", weight: 5}`
- Restart Sapphire after editing to recompute centroids
- Run `python test_classifier.py` to verify accuracy (target: 100%)

## Troubleshooting

### Sapphire won't start

**"Model not loaded" or connection refused:**

- fastembed auto-downloads the model on first run (~100 MB). Ensure internet access.
- Check `~/cache/huggingface/` for downloaded models.
- Run `pip install -r requirements.txt` to ensure all deps are installed.

### Bot gets no response

**Check each layer from bottom up:**

- `curl http://127.0.0.1:3124/health` -- Krystal must respond
- `curl http://127.0.0.1:3123/health` -- Sapphire must respond
- `curl http://127.0.0.1:3126` -- Emerald must respond (WebSocket upgrade)
- Bot config must point `emerald_host: localhost` + `emerald_port: 3126`
- Check `pm2 logs` for each service

### Krystal crashes or OOM

- Reduce `KRYSTAL_N_CTX` (try 2048)
- Use a smaller quantized model (Q4_K_M is already small)
- Reduce `KRYSTAL_N_THREADS` to match available cores
- Check `pm2 logs krystal`

### Emerald not receiving bot messages

- Check Emerald logs for WebSocket connection events
- Verify `emerald_host`/`emerald_port` in bot config.yml
- Ensure Emerald started before the bot
