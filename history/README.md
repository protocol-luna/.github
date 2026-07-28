# Architecture History

This folder traces the evolution of Luna Protocol — from a single monolithic binary to a modular ecosystem of six services.

```
▰ Monolith               Jade + Krystal      + Pixieglow         + Sapphire         + Emerald          + Ruby
│                        │                   │                   │                  │                  │
│ 01                     02                  03                  04                 05                 06
│                        │                   │                   │                  │                  │
│ one binary             two services        three services       four services     five services      six services
│ Discord only           Discord only        Discord + Matrix    + classification  + brain mutualized  + spontaneity
│ no HTTP                :3124               :3124               :3123-3125         :3123-3127         :3123-3127
└──────────────────────────────────────────────────────────────────────────────────────────────────────▶
```

## The story

| Step | File | What happened |
|------|------|---------------|
| **1** | [`01-luna-client-server.md`](01-luna-client-server.md) | First version. One codebase, one process. Discord bot + llama.cpp inference bundled together. No HTTP, no WebSocket. |
| **2** | [`02-jade-krystal.md`](02-jade-krystal.md) | The monolith splits. **Jade** becomes the Discord client; **Krystal** becomes the dedicated inference server (`:3124`). First separation of concerns. |
| **3** | [`03-pixieglow.md`](03-pixieglow.md) | **Pixieglow** joins as a Matrix adapter. LUNA goes multiplatform. But behavior logic is duplicated across both adapters. |
| **4** | [`04-sapphire.md`](04-sapphire.md) | **Sapphire** enters as an LLM gateway — it classifies messages (intéressant/futile), manages sessions, constructs prompts, detects degenerate outputs. Krystal splits into two instances: small model (1.5B) and large model (8B). |
| **5** | [`05-emerald.md`](05-emerald.md) | **Emerald** becomes the central brain. It mutualizes all behavior, routing, and state from the adapters. Jade and Pixieglow become thin adapters. WebSocket replaces HTTP for adapter ↔ brain communication. |
| **6** | [`06-ruby.md`](06-ruby.md) | **Ruby** adds a Markov chain service for spontaneous, context-free message generation. Zero latency, no model loading. The architecture is complete. |

## How to read

Each file contains:
- A **mermaid diagram** showing the architecture at that point in time
- The **key change** introduced at that step
- A brief description of why the change was made

Start at `01` and read chronologically, or jump to any step to see what the system looked like.

## Final architecture

```
Platform → Adapter (WS :3126) → Emerald → Sapphire (:3123) → Krystal (:3124-3125)
                                         ↘ Ruby (:3127)
```

6 services, 5 repos, 2 platforms, 1 brain.
