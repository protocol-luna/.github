# Few-Shot Priming Guide

## What is Few-Shot Priming?

Few-shot priming is an LLM technique where you provide a few examples of the desired behavior before the actual task. Instead of only relying on system prompts, the model learns patterns from concrete examples of conversations.

### Example

```
System Prompt: "You are a casual Discord bot"

Few-Shot Examples:
User: "yo whats good"
Bot: "nm just chillin, u"

User: "bored af"
Bot: "lol same energy fr"

[Real message comes here]
User: "hey how are you"
Bot: [generates response in the same casual style]
```

## Why Use Few-Shot Priming?

### Benefits

1. **Style Consistency**: The model learns the exact tone and register you want
2. **Response Patterns**: Examples show preferred message length and structure
3. **Context Understanding**: Models better understand what kind of responses you expect
4. **Better Performance**: Especially effective with smaller models like Luna-Protocol-1.5B
5. **Reduced System Prompt Complexity**: Show instead of tell

### Why Few-Shot is More Efficient with Luna-Protocol

The Luna-Protocol model is a fine-tuned Qwen2.5 1.5B trained on only **200k samples** from the Discord-Dialogues dataset. This limited training data means:

- **Greater flexibility**: The model hasn't overfit to specific conversation patterns, making it more responsive to in-context examples
- **Faster adaptation**: Few-shot priming can redirect the model's behavior more effectively than with models trained on massive datasets
- **Improved consistency**: Examples provide clear signals that the model can quickly align with

### When to Use

- ✅ **Small to medium models** (< 7B parameters) -- huge impact
- ✅ **Discord bots** -- patterns vary widely, examples help a lot
- ✅ **Personality-driven characters** -- consistency is key
- ✅ **Specialized conversation styles** -- technical, casual, formal, etc.
- ⚠️ **Large models** (70B+) -- less necessary but still helps
- ⚠️ **Simple tasks** -- might add unnecessary tokens

## Configuration

### Basic Setup (Sapphire server-side)

Few-shot is handled server-side by Sapphire (Python), not in the bot config. Edit Sapphire's `few_shot_examples.yml` file:

```yaml
# few_shot_examples.yml
- user: "hello"
  assistant: "hey, what's up?"
- user: "how are you"
  assistant: "doing good tbh, just chilling"
```

Or set the path via Sapphire's config:

```yaml
# Sapphire config.yml
few_shot_enabled: true
few_shot_examples_path: ./few_shot_examples.yml
```

### Disabling

```yaml
# Sapphire config.yml
few_shot_enabled: false
```

## Best Practices

### 1. Quality Over Quantity

**Good**: 3-5 high-quality examples
```yaml
few_shot_examples:
  - user: "yo"
    assistant: "sup"
  - user: "whats up"
    assistant: "nm, hbu"
```

**Bad**: 20 mediocre examples (wastes tokens, dilutes pattern)

### 2. Match Your Bot's Voice

**Casual bot**:
```yaml
few_shot_examples:
  - user: "hey luna"
    assistant: "yooo sup"
```

**Professional bot**:
```yaml
few_shot_examples:
  - user: "Hello, can you help?"
    assistant: "Of course, I'm happy to assist."
```

**Sarcastic bot**:
```yaml
few_shot_examples:
  - user: "are you smart"
    assistant: "yeah sure, whatever makes you sleep at night lol"
```

### 3. Consistent Message Length

If you want brief responses, keep examples brief:
```yaml
few_shot_examples:
  - user: "hi"
    assistant: "hey"
  - user: "what's up"
    assistant: "nm"
```

If you want detailed responses:
```yaml
few_shot_examples:
  - user: "tell me about yourself"
    assistant: "i'm a discord bot powered by local LLMs. i love chatting with people and just vibing in communities."
```

### 4. Vary the Examples

Include different types of interactions:
```yaml
few_shot_examples:
  # Greetings
  - user: "hey"
    assistant: "yo what's good"
  # Questions
  - user: "how are you"
    assistant: "doing pretty good ngl"
  # Statements
  - user: "that's cool"
    assistant: "fr fr"
  # Follow-ups
  - user: "lol that's funny"
    assistant: "ikr"
  # Expressions
  - user: "i'm bored"
    assistant: "same energy honestly"
```

### 5. Realism

Examples should sound natural, like real conversations:

**Good**:
```yaml
- user: "yo whats good"
  assistant: "nm just chillin, u"
```

**Weird/Robotic**:
```yaml
- user: "I would like to inquire about your status"
  assistant: "AFFIRMATIVE: I AM FUNCTIONING AT OPTIMAL CAPACITY"
```

## Technical Details

### How It Works in Luna Protocol

1. On startup, Sapphire loads `few_shot_examples.yml` via `load_few_shot_examples()`
2. On each `/v1/respond` request, `format_few_shot_examples()` converts them to message objects
3. `inject_few_shot_into_conversation()` inserts them after the system prompt, before conversation history
4. Model receives: `[system_prompt] + [examples] + [conversation_history]`

Bots are thin SSE clients -- they send `{ username, text, session_id }` to Sapphire and receive streamed tokens. No few-shot logic runs in the bot.

### Token Usage

- Each example adds ~10-30 tokens to your context window
- With prompt caching (llama-server `--cache-reuse`), examples are cached after first request
- Minimal impact on subsequent messages in the same session

### Code Integration

In Sapphire's `src/sapphire/few_shot.py`:

```python
def inject_few_shot_into_conversation(
    messages: list[dict],
    few_shot_messages: list[dict],
) -> list[dict]:
    if not messages:
        return list(few_shot_messages)
    system, *rest = messages
    return [system, *few_shot_messages, *rest]
```

## Example Configurations

### Luna-Protocol Bot (Sapphire `few_shot_examples.yml`)

```yaml
- user: "yo whats good"
  assistant: "nm just chillin, u"
- user: "bored af"
  assistant: "lol same energy fr"
- user: "hey how are you"
  assistant: "im doing pretty good tbh, just vibing"
- user: "whats up"
  assistant: "yooo not much, what about you"
- user: "how was your day"
  assistant: "it was alright, nothing crazy happened lol"
```

### Help-Focused Bot

```yaml
- user: "how do i do this"
  assistant: "hmm usually you can just... try restarting first?"
- user: "thanks that worked"
  assistant: "nice! glad i could help"
- user: "one more question"
  assistant: "sure go ahead, what's up"
```

### Formal Assistant

```yaml
- user: "Hello, can you assist?"
  assistant: "Of course, I'd be happy to help. What do you need?"
- user: "Thank you for your time"
  assistant: "You're very welcome. Please don't hesitate to reach out again."
```

## Troubleshooting

### Issue: Bot ignores few-shot examples

**Check**:
- `few_shot_enabled: true` in Sapphire's config
- Examples are properly formatted with `user` and `assistant` keys in `few_shot_examples.yml`
- Restarted Sapphire after editing examples
- Check Sapphire startup logs for `"N few-shot examples (enabled=True)"`

### Issue: Bot sounds robotic or inconsistent

**Solution**:
- Make examples more natural and conversational
- Ensure examples match your desired tone
- Reduce number of examples to 3-4
- Check system prompt doesn't contradict examples

### Issue: Too much token usage

**Solution**:
- Reduce number of examples (3-5 is optimal)
- Examples are cached after first message, minimal impact after
- Use shorter examples

### Issue: Changes not taking effect

**Solution**:
- Few-shot examples are loaded at Sapphire startup -- restart Sapphire after editing `few_shot_examples.yml`
- Set `few_shot_enabled: false` in config to disable temporarily
- Check Sapphire logs for load errors on startup

## Performance Impact

### Token Overhead

- 5 examples = ~50-150 tokens
- Minimal with prompt caching (llama-server)
- Negligible after first message in session

### Model Performance

| Model Size | Impact |
|---|---|
| Small (1.5B) | ⭐⭐⭐⭐⭐ Significant improvement |
| Medium (7B) | ⭐⭐⭐⭐ Good improvement |
| Large (13B+) | ⭐⭐⭐ Modest improvement |
| Very Large (70B+) | ⭐⭐ Minimal improvement |

## Related Resources

- [OpenAI Few-Shot Learning](https://platform.openai.com/docs/guides/prompt-engineering/few-shot-learning)
- [Luna Protocol -- Sapphire Few-Shot Module](../sapphire/src/sapphire/few_shot.py)
- [Sapphire Config](../sapphire/config.example.yml)
- [Example Few-Shot File](../sapphire/few_shot_examples.yml)
