# Centroid Classification & Emotional State

How Sapphire uses embedding centroids to route messages, track emotion, and adjust LLM parameters -- all without a single neural classifier.

## What Is a Centroid?

A **centroid** is the geometric center of a set of points. In Sapphire, each point is a **384-dimensional embedding vector** produced by the `BAAI/bge-small-en-v1.5` model -- a lightweight, CPU-friendly embedding model that converts any text into a fixed-size numerical vector representing its semantic meaning.

The idea is simple: gather dozens of example messages for a category, embed each one, and run **k-means clustering** (k=10, seed=42) to find 10 representative centroids per class. These centroids capture the natural subtypes within each category -- for example, the "futile" class includes greetings, farewells, affirmations, and small talk, each forming its own cluster.

Classifying a new message then becomes a distance problem instead of a learning problem:

```python
def _kmeans(data, k=10, max_iters=20):
    """Returns (k, dim) centroids via k-means with fixed seed 42."""
    n, dim = data.shape
    k = min(k, n)
    rng = np.random.default_rng(42)
    idx = rng.choice(n, k, replace=False)
    centroids = data[idx].copy()
    for _ in range(max_iters):
        dists = np.cdist(data, centroids)
        labels = dists.argmin(axis=1)
        new = np.array([data[labels == i].mean(axis=0)
                        for i in range(k)])
        if np.allclose(centroids, new):
            break
        centroids = new
    return centroids

def classify(text, embedder, futile_centroids, interessant_centroids):
    emb = embedder.query_embed(text)               # 384-D vector
    sim_futile = max(cosine_similarity(emb, c)
                     for c in futile_centroids)     # best match across 10
    sim_inter = max(cosine_similarity(emb, c)
                    for c in interessant_centroids)  # best match across 10
    delta = sim_inter - sim_futile
    label = "INTERESSANT" if delta > 0 else "FUTILE"
    return label, abs(delta), sim_futile, sim_inter
```

The closest centroid wins, but unlike a single centroid, the multicentroid approach captures sub-types within each class -- a greeting and a farewell both land near one of the 10 futile centroids even though they're far from each other in embedding space. No training, no gradient descent, no GPU required -- just k-means at startup and dot products at runtime.

## Why DistilBERT Couldn't Help

Before centroids, Sapphire used a **DistilBERT classifier** fine-tuned on the same example data. DistilBERT is a distilled version of BERT -- smaller and faster, but still a full transformer model. It seemed like the right tool: a neural classifier should be more accurate than a simple centroid, right?

In practice, it failed for three reasons:

### 1. Training Instability

DistilBERT needs a proper train/validation split, label-balanced batches, and early stopping -- or it overfits badly. With only ~1000 examples and a model of 66 million parameters, overfitting was the default behavior. Every time I added or removed examples, I had to re-tune hyperparameters. The centroid approach has zero hyperparameters -- you just compute means.

### 2. Cold Start for New Categories

Adding a new classification category (like the emotion centroids that came later) meant labeling examples, re-training the entire DistilBERT model, and hoping the decision boundary didn't shift for old categories. With centroids, adding a category is a single line of code: `centroids["new_category"] = mean(embed(examples))`.

### 3. Opaque Decisions

A DistilBERT classifier outputs a probability distribution over classes. When it misclassifies something, you can't easily tell *why*. With centroids, you can inspect the nearest examples, look at the raw similarity scores, and see exactly which examples are pulling the centroid toward a given region. The decision boundary is transparent by construction.

Centroids trade a small amount of accuracy (in theory) for massive gains in **maintainability, interpretability, and flexibility**. In practice, the accuracy gap didn't materialize -- the centroid classifier matches DistilBERT's F1 score on our data, because the embedding space already separates the categories well.

> **Key insight:** A centroid classifier doesn't learn a decision boundary -- it inherits one from the embedding model. The quality of the classification depends almost entirely on the quality of the embedding space, not on the classifier itself. This is why choosing `bge-small-en-v1.5` (a model trained on 1.1B training pairs specifically for semantic similarity) was more important than any classifier architecture.

## FUTILE vs INTERESSANT

Sapphire maintains two classification centroid sets (10 each via k-means):

| Centroid | Examples | Count | LLM Backend |
|---|---|---|---|
| `futile` | "lol", "ok", "hello", "nm just chillin u", "sup", "nice" | ~580 | Luna 1.5B (fast, :3124) |
| `interessant` | "how does the fine-tuning work?", "i feel really sad today", "what do you think about...", technical questions, confidences | ~545 | Hermes 3B/8B (deep, :3125) |

When a message arrives, Sapphire computes the maximum cosine similarity across all 10 centroids per class:

```python
sim_futile = max(cos(emb, c) for c in futile_centroids)         # 10 centroids
sim_inter  = max(cos(emb, c) for c in interessant_centroids)     # 10 centroids
diff = sim_inter - sim_futile

if diff > 0:
    label = "INTERESSANT"
    backend = KRYSTAL_SEMANTIC  # port 3125
else:
    label = "FUTILE"
    backend = KRYSTAL_GENERIC    # port 3124
```

The `diff` value also measures **confidence**. A diff of +0.31 means the message is clearly interesting; a diff of -0.02 means it's barely futile. In practice, about **5.8% of messages** fall within the ambiguous zone where the classifier isn't confident (diff close to zero).

The multicentroid approach achieves **100% accuracy on the 396-example test suite** with the seed fixed at 42, ensuring reproducible centroids across rebuilds.

## Visualizing the Centroid Space

The 384-dimensional embedding space can be projected down to 3 dimensions using PCA for visualization. The **10 diamond markers per class** (20 total) are the k-means centroids (k=10, seed=42).

Blue points are futile examples, yellow points are interesting ones. Hover over any point to see the original text, similarity scores, and classification.

The annotation shows the exact counts: 579 futile examples, 545 interesting examples, and the total explained variance of the PCA projection.

## Valence and Arousal

Classification is only half of what centroids do. Sapphire uses **the exact same k-means multicentroid mechanism** on a second, independent set of centroids to measure the emotional content of every message. Each emotion pole also has 10 centroids computed via k-means (k=10, seed=42), and valence/arousal uses the maximum similarity across all 10:

```python
def _max_sim(emb, centroids):
    return max(cosine_similarity(emb, c) for c in centroids)

valence = _max_sim(emb, positive_centroids) - _max_sim(emb, negative_centroids)
arousal = _max_sim(emb, high_arousal_centroids) - _max_sim(emb, low_arousal_centroids)
```

| Axis | Positive pole | Negative pole |
|---|---|---|
| **Valence** | "hell yeah", "love that", "this is great", "amazing" | "shut up", "i hate this", "this sucks", "terrible" |
| **Arousal** | "WHAT THE HELL", "omg omg omg", "AAAAA", "NO WAY" | "just chilling", "meh", "i guess", "whatever" |

Both values range from **-1.0 to +1.0**. Together they form the **circumplex model of affect** (Russell, 1980) -- the same psychological framework that inspired the PARRY chatbot in 1972.

The valence/arousal pair tells us not just *what* someone said, but *how* they said it: a calm "i hate this" (negative valence, low arousal) is different from an explosive "I HATE THIS" (negative valence, high arousal), and both are different from an excited "hell yeah!" (positive valence, high arousal).

## Emotional State Persistence

Raw valence and arousal from a single message are noisy. One angry outburst shouldn't define the entire conversation. Sapphire solves this with an **exponential moving average (EMA)** that maintains a smoothed emotional state per session:

```python
class EmotionState:
    def __init__(self, decay=0.85, deadzone=0.005):
        self.decay = decay
        self.deadzone = deadzone

    def update(self, key, valence_delta, arousal_delta):
        if abs(valence_delta) < self.deadzone:
            valence_delta = 0.0
        if abs(arousal_delta) < self.deadzone:
            arousal_delta = 0.0
        s = self._states.setdefault(session_key,
                                     {"valence": 0.0, "arousal": 0.0})
        # EMA: 85% old state, 15% new signal
        s["valence"] = s["valence"] * self.decay \
                       + valence_delta * (1 - self.decay)
        s["arousal"] = s["arousal"] * self.decay \
                       + arousal_delta * (1 - self.decay)
        return s
```

| Concept | Mathematical expression | What it means |
|---|---|---|
| State at turn N | `E_N` | The emotional state after processing N messages |
| Decay | `λ = 0.85` | 85% of the previous state is preserved each turn |
| Input at turn N+1 | `e_{N+1}` | Raw valence/arousal of the new message |
| State at turn N+1 | `E_{N+1} = λE_N + (1-λ)e_{N+1}` | The new state blends old and new |
| Half-life | `t_{1/2} = ln(0.5)/ln(0.85) ≈ 4.3` | A spike decays to half its influence after ~4 messages |
| Steady state | `E_∞ ≈ μ` | After many messages, the state converges to the mean input |

The **deadzone** (0.005) is a threshold: small emotional fluctuations (a mildly positive "ok") are treated as neutral. This prevents gradual drift -- the state only moves when the signal is strong enough to matter.

### Why this matters for n+1 transitivity

Because the state is an EMA, **the order of messages matters**. Consider two sequences:

```
Sequence A:  sad, sad, sad, happy, happy, happy
Sequence B:  happy, happy, happy, sad, sad, sad
```

After all 6 messages, the aggregate valence is the same (3 sad + 3 happy = neutral). But with the EMA, **Sequence A ends in a positive state** (the recent happy messages dominate) while **Sequence B ends in a negative state** (the recent sad messages dominate). The state at turn N+1 depends on both the new input and the entire trajectory of previous states, weighted by recency.

This transitivity property is what makes the bot "feel" like it has a mood: an insult at message 2 still colors the bot's responses at message 10, but by message 20 it's faded. A series of positive exchanges slowly lifts the bot's mood, regardless of what started the conversation.

The state is scoped by **session key** (derived from the conversation's id_slot in llama.cpp), so different conversations maintain independent emotional states. The bot can be furious in one thread and cheerful in another.

## How Emotion Controls the LLM

The emotional state doesn't just sit there -- it actively controls the LLM's sampling parameters.

### Temperature follows arousal

```python
temperature = clamp(0.7 + arousal * 0.3, 0.4, 1.0)
```

| Arousal | Temperature | Effect on responses |
|---|---|---|
| -1.0 (calm) | 0.40 | Predictable, repetitive, "tired" answers |
| 0.0 (neutral) | 0.70 | Default creativity, normal variation |
| +1.0 (excited) | 1.00 | Unpredictable, creative, "hyped" answers |

When the conversation is heated (high arousal), the model produces more varied and surprising responses -- like a real person who gets more animated. When it's calm, responses are measured and predictable.

### Repeat penalty follows valence

```python
repeat_penalty = clamp(1.15 - valence * 0.1, 1.0, 1.3)
```

| Valence | Repeat Penalty | Effect on responses |
|---|---|---|
| -1.0 (negative) | 1.25 | Avoids repetition forcefully, "searching for words" |
| 0.0 (neutral) | 1.15 | Default repetition control |
| +1.0 (positive) | 1.05 | Allows casual repetition, "chatty" mode |

Negative conversations get a higher repeat penalty -- the model avoids falling into repetitive loops, mimicking someone who's carefully choosing their words in a tense exchange. Positive conversations allow more natural repetition, like casual banter.

## Practical Impact

The centroid-based system has been running in production for months. The key numbers:

- **~70% of messages** are classified FUTILE and handled by the fast 1.5B model (average response: ~400ms)
- **~30% of messages** are classified INTERESSANT and routed to the larger model (~800ms average)
- **~5.8% of messages** fall in the ambiguous zone (diff near zero) -- still handled correctly, just with lower confidence
- **Zero classifier retraining** since deployment: adding examples to a centroid means appending to a YAML file, not re-running training
- **~250ms for a full classification + emotion pass** on CPU (embeddings only, no GPU required)
- **100% accuracy** on the 396-example test suite with k-means multicentroid (k=10)

The centroid approach turned a machine learning problem into a geometry problem -- and geometry is easier to debug, maintain, and improve.
