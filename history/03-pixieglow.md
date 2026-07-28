# 3. Pixieglow Joins (Multi-Platform)

**Pixieglow**, the Matrix adapter, is added. It follows the same pattern as Jade: connects to a platform (Matrix) and talks directly to Krystal for inference. No shared brain yet — each adapter manages its own logic.

```mermaid
flowchart LR
    classDef platform fill:#2c3e50,color:#fff,stroke-width:2px
    classDef jade fill:#1abc9c,color:#fff,stroke-width:2px
    classDef pixie fill:#e84393,color:#fff,stroke-width:2px
    classDef inference fill:#8e44ad,color:#fff,stroke-width:2px

    Discord["Discord"]:::platform
    Matrix["Matrix"]:::platform
    Jade["Jade (Discord)"]:::jade
    Pixieglow["Pixieglow (Matrix)"]:::pixie
    Krystal["Krystal (llama.cpp)
    :3124"]:::inference

    Discord --> Jade
    Matrix --> Pixieglow
    Jade -- HTTP --> Krystal
    Pixieglow -- HTTP --> Krystal
```

**Key change**: LUNA is now multiplatform. But each adapter duplicates behavior logic — Jade and Pixieglow both implement their own response routing, state management, and prompt construction. Maintaining feature parity across two codebases is becoming painful.
