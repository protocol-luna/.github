# 5. Emerald (Brain Service)

**Emerald** is introduced as the central brain. It mutualizes all the behavior, routing, and state management that was previously duplicated across Jade and Pixieglow. The adapters become **pure adapters** — they forward platform events to Emerald over WebSocket and post responses back. No more "thinking" in the adapters.

Sapphire is renamed from "LLM Gateway" to a classification + prompt service. The two Krystal instances are formalized: **Krystal (Small)** for `FUTILE` messages, **Krystal (Large)** for `INTERESSANT` messages.

```mermaid
flowchart LR
    classDef platform fill:#2c3e50,color:#fff,stroke-width:2px
    classDef jade fill:#1abc9c,color:#fff,stroke-width:2px
    classDef pixie fill:#e84393,color:#fff,stroke-width:2px
    classDef brain fill:#27ae60,color:#fff,stroke-width:2px
    classDef service fill:#2980b9,color:#fff,stroke-width:2px
    classDef inference fill:#8e44ad,color:#fff,stroke-width:2px

    Discord["Discord"]:::platform
    Matrix["Matrix"]:::platform
    Jade["Jade (Discord Adapter)"]:::jade
    Pixieglow["Pixieglow (Matrix Adapter)"]:::pixie
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

    Discord --> Jade
    Matrix --> Pixieglow
    Jade -- ":3126 WS" --> Emerald
    Pixieglow -- ":3126 WS" --> Emerald
    Emerald -- ":3123 HTTP" --> Sapphire
    Sapphire -- ":3124" --> KrystalS
    Sapphire -- ":3125" --> KrystalL
```

**Key change**: Architecture is now cleanly layered. Platform → Adapter (WS) → Emerald → Sapphire → Krystal. Adapters are thin (<300 lines of real logic). All behavior lives in Emerald. WebSocket replaces HTTP for adapter ↔ brain communication to enable streaming.
