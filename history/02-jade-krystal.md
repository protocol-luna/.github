# 2. Jade + Krystal (Monolith Split)

The monolith is split into two services. **Jade** becomes the Discord client (named after the green gem). **Krystal** becomes the dedicated llama.cpp inference server. No concept of "small" vs "large" model yet — Krystal is just one server.

```mermaid
flowchart LR
    classDef platform fill:#2c3e50,color:#fff,stroke-width:2px
    classDef jade fill:#1abc9c,color:#fff,stroke-width:2px
    classDef inference fill:#8e44ad,color:#fff,stroke-width:2px

    Discord["Discord"]:::platform
    Jade["Jade (Discord Client)"]:::jade
    Krystal["Krystal (llama.cpp)
    :3124"]:::inference

    Discord --> Jade
    Jade -- HTTP --> Krystal
```

**Key change**: Separation of concerns. Jade handles Discord events and response formatting; Krystal handles LLM inference. Jade can now be restarted without dropping the model from RAM. First step toward a multi-platform architecture.
