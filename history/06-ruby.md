# 6. Ruby (Markov Chain Spontaneity)

**Ruby** is added as a lightweight Markov chain service (order-4, SQLite-backed). It generates spontaneous, context-free messages from real conversation data. Emerald calls Ruby for random triggers or when the bot hasn't spoken in a while.

This completes the architecture. Ruby is optional (dashed line), purely spontaneous — no prompt, no model loading, zero latency.

```mermaid
flowchart TD
    classDef platform fill:#2c3e50,color:#fff,stroke-width:2px
    classDef jade fill:#1abc9c,color:#fff,stroke-width:2px
    classDef pixie fill:#e84393,color:#fff,stroke-width:2px
    classDef brain fill:#27ae60,color:#fff,stroke-width:2px
    classDef service fill:#2980b9,color:#fff,stroke-width:2px
    classDef inference fill:#8e44ad,color:#fff,stroke-width:2px
    classDef alt fill:#c0392b,color:#fff,stroke-width:2px

    Discord["Discord"]:::platform
    Matrix["Matrix"]:::platform

    Jade["Jade (Discord Adapter)"]:::jade
    Pixieglow["Pixieglow (Matrix Adapter)"]:::pixie
    Discord ~~~ Matrix

    Emerald["Emerald (Brain)
    behavior · triggers · routing"]:::brain

    Sapphire["Sapphire (LLM Gateway)
    classify · emotion · sessions"]:::service

    KrystalS["Krystal (Small)
    Luna 1.5B · :3124
    FUTILE messages"]:::inference

    KrystalL["Krystal (Large)
    Discord-Hermes-8B · :3125
    INTERESSANT messages"]:::inference

    Ruby["Ruby (Markov Chain)
    order-4 · SQLite · 0 latency"]:::alt

    Discord --> Jade
    Matrix --> Pixieglow
    Jade -- ":3126 WS" --> Emerald
    Pixieglow -- ":3126 WS" --> Emerald
    Emerald -- ":3123 HTTP" --> Sapphire
    Sapphire -- ":3124" --> KrystalS
    Sapphire -- ":3125" --> KrystalL
    Emerald -.->|":3127"| Ruby
```

**Final architecture**: 6 services, 5 repos (Krystal is one service with two instances), 2 platforms. Emerald is the single source of truth for behavior. Every service has a single responsibility.
