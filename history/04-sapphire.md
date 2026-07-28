# 4. Sapphire (LLM Gateway & Classification)

**Sapphire** is introduced as a dedicated LLM gateway service. It handles:
- **Classification** — marking messages as `intéressant` (interesting) or `futile` (pointless) via embedding-based centroid matching
- **Session management** — per-channel conversation history
- **Prompt construction** — few-shot example injection, emotion-aware formatting
- **Degenerate output detection** — filters bad responses, triggers retries

Krystal now runs **two instances**: a small model (1.5B for `futile` messages) and a large model (8B for `intéressant` messages). But the small/large naming convention is not yet formalized.

```mermaid
flowchart LR
    classDef platform fill:#2c3e50,color:#fff,stroke-width:2px
    classDef jade fill:#1abc9c,color:#fff,stroke-width:2px
    classDef pixie fill:#e84393,color:#fff,stroke-width:2px
    classDef service fill:#2980b9,color:#fff,stroke-width:2px
    classDef inference fill:#8e44ad,color:#fff,stroke-width:2px

    Discord["Discord"]:::platform
    Matrix["Matrix"]:::platform
    Jade["Jade (Discord)"]:::jade
    Pixieglow["Pixieglow (Matrix)"]:::pixie
    Sapphire["Sapphire (LLM Gateway)
    classify · sessions · prompt"]:::service
    Krystal1["Krystal (Small)
    1.5B · :3124"]:::inference
    Krystal2["Krystal (Large)
    8B · :3125"]:::inference

    Discord --> Jade
    Matrix --> Pixieglow
    Jade -- HTTP --> Sapphire
    Pixieglow -- HTTP --> Sapphire
    Sapphire -- ":3124" --> Krystal1
    Sapphire -- ":3125" --> Krystal2
```

**Key change**: Classification logic is extracted from the adapters into a shared service. Both Jade and Pixieglow now talk to Sapphire instead of directly to Krystal. The two-tier inference (small vs. large model) appears for the first time. But the adapters still contain behavior/routing logic — they're not yet pure adapters.
