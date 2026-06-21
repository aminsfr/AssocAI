---
title: Prompt Engineering
nav_order: 10
difficulty: 🟢
---

# Prompt Engineering

## Intro

Prompt engineering is the practice of structuring what you send to a model so it reliably does what you actually want — not just once, but every time you run it. If you've used ChatGPT, you've already been doing a rough version of this. The difference when you're building with the API is that there's no chat history smoothing things over and no opportunity to just rephrase and try again in the moment — your prompt has to work standalone, often inside code that runs unattended. Skip this and everything downstream (tool calling, agents, RAG) inherits the same unreliability.

You don't need this if you're doing a one-off question in a chat UI — trial and error is fine there. You need this the moment a prompt is going to run more than once, run without you watching it, or feed into something else.

## Concept

A prompt is not "a sentence you type." When you call an API, what the model actually sees is closer to a small program made of distinct parts:

- **System prompt** — sets the model's role, constraints, and tone. This is configuration, not conversation. It rarely changes between calls.
- **User message** — the actual task or question for this specific call.
- **Context** — any extra information the model needs (a document, prior turns, retrieved data).
- **Examples** (optional) — sample inputs/outputs that show the model the pattern you want, instead of just describing it.

Most prompting problems come down to one of two failures: the model didn't have the information it needed, or it had the information but wasn't told clearly enough what to do with it. Good prompt engineering is mostly about eliminating ambiguity, not finding magic phrases.

A few principles that consistently move the needle:

- **Be explicit about format.** "Summarize this" gets you prose. "Summarize this in 3 bullet points, each under 15 words" gets you something you can actually use in code.
- **Show, don't just tell.** One good example of the input/output pattern you want usually beats a paragraph of instructions describing it.
- **Separate instructions from data.** Mixing "do this" with "here's the content" in one undifferentiated blob is the single most common source of the model getting confused about what's an instruction versus what's content to process.
- **Ask for reasoning before the answer, when the task is non-trivial.** Having the model think through a problem step by step before producing the final output measurably improves accuracy on anything that isn't a lookup.
- **Iterate like code, not like conversation.** Change one variable, re-run, compare. Don't keep stacking caveats onto a prompt that isn't working — start the structure over.

## Hands-on

Here's the difference between a vague prompt and an engineered one, using the OpenAI API in Python.

**Vague:**

```python
from openai import OpenAI

client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "user", "content": "Tell me about this customer review: 'Shipping took forever but the product is great, would buy again.'"}
    ],
)

print(response.choices[0].message.content)
```

This works, technically. But the output format is unpredictable — sometimes a paragraph, sometimes bullets, sometimes it adds commentary you didn't ask for. Unusable if you need to parse the result in code.

**Engineered:**

```python
from openai import OpenAI

client = OpenAI()

system_prompt = """You are a customer feedback classifier. Given a review, extract:
- sentiment: one of "positive", "negative", "mixed"
- key_issues: a list of specific problems mentioned (empty list if none)
- key_praise: a list of specific things praised (empty list if none)

Respond with ONLY valid JSON in this exact shape, no other text:
{"sentiment": "...", "key_issues": [...], "key_praise": [...]}"""

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": "Shipping took forever but the product is great, would buy again."}
    ],
    temperature=0,
)

print(response.choices[0].message.content)
# {"sentiment": "mixed", "key_issues": ["slow shipping"], "key_praise": ["product quality"]}
```

What changed: the task moved from a vague ask to a defined contract — fixed fields, fixed types, an explicit "only JSON" instruction, and `temperature=0` to reduce run-to-run variance. This version is something you can actually pipe into a database or a UI without babysitting it.

## Common pitfalls

- **Treating the prompt as "done" after it works once.** Test it against a handful of varied real inputs, not just the example you wrote it around.
- **Burying the actual instruction in a wall of context.** If the task is at the bottom of a long block of text, it's easy for the model to under-weight it. Put the core instruction up front or repeat it at the end.
- **Asking for JSON without enforcing it.** Plain prompting can still produce malformed JSON occasionally. For anything that needs to be reliably parseable, use the API's structured output / JSON schema features rather than relying on the prompt alone — that's its own article, coming soon.

## Resources

- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering) — the most practically dense official guide, written by the people whose models you're prompting.
- [Anthropic Prompt Engineering Docs](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview) — strong on structured tags and step-by-step reasoning techniques, transfers well beyond just Claude.
- [Learn Prompting](https://learnprompting.org/) — free, community-maintained, good if you want a broader survey of named techniques (few-shot, chain-of-thought, etc.) in one place.
- [OpenAI Cookbook](https://github.com/openai/openai-cookbook) — real, runnable code examples instead of theory, useful once you're past the basics here.
