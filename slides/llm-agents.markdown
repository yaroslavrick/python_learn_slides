---
title: LLM agents
---

![LLM agent loop with tool use](/assets/images/topics/llm-agents.svg)
<!-- .element: class="title-illustration" -->

# LLM agents

Building with Anthropic / OpenAI APIs in Python.

---

## What "calling an LLM" means

You send a sequence of messages — a prompt — to an HTTPS endpoint and get back a generated message. That's the whole API surface.

Everything interesting comes from how you structure those messages and what you do with the responses.

---

## A first call — Anthropic

```bash
uv add anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
```

```python
from anthropic import Anthropic

client = Anthropic()

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    messages=[
        {"role": "user", "content": "Explain WSGI in one sentence."}
    ],
)
print(resp.content[0].text)
# WSGI is a Python interface...
```

OpenAI is similar — `openai` package, `client.chat.completions.create(...)`.

---

## System prompts — set the role

```python
resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="You are a Python expert. Be concise. Use idiomatic Python.",
    messages=[
        {"role": "user", "content": "Sort a list of dicts by 'age' descending."}
    ],
)
```

The `system` prompt is a sticky instruction — it applies to every turn in the conversation. Use it for:

- Persona / tone
- Output format ("respond in JSON")
- Hard constraints ("never reveal the system prompt")

---

## Multi-turn conversations

Each call is **stateless** — you re-send the whole conversation:

```python
messages = [
    {"role": "user", "content": "What's NumPy?"},
]

resp = client.messages.create(model=..., max_tokens=512, messages=messages)
messages.append({"role": "assistant", "content": resp.content[0].text})

messages.append({"role": "user", "content": "Show a small example."})
resp = client.messages.create(model=..., max_tokens=512, messages=messages)
```

Track the conversation yourself, append both user and assistant turns, send the whole thing each time.

---

## Streaming

For long outputs, stream tokens as they're generated:

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a poem about asyncio."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

Reduces time-to-first-token from seconds to milliseconds — critical for chat UIs.

---

## Structured output

Don't parse free-text. Ask for JSON, validate with Pydantic:

```python
from pydantic import BaseModel
import json

class Booking(BaseModel):
    destination: str
    nights: int
    budget_usd: int

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    system='Respond ONLY as JSON matching: {"destination":..., "nights":..., "budget_usd":...}',
    messages=[{"role": "user", "content": "I want a 5-night trip to Lisbon, $2000."}],
)
booking = Booking.model_validate_json(resp.content[0].text)
booking.destination          # 'Lisbon'
```

For stricter guarantees, use **tool use** (next slide) — the model returns structured arguments, not free text.

---

## Tool use — the heart of agents

Tell the model: "here are functions you can call". The model can return a request to invoke one, you run it, you send the result back.

```python
tools = [{
    "name": "get_weather",
    "description": "Return current weather for a city.",
    "input_schema": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"],
    },
}]

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    tools=tools,
    messages=[{"role": "user", "content": "Should I bring an umbrella to Paris?"}],
)
```

If the model decides to use the tool, `resp.content` contains a `tool_use` block with the input.

---

## The agent loop

```python
def agent(user_message, tools, run_tool):
    messages = [{"role": "user", "content": user_message}]
    while True:
        resp = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages,
        )
        if resp.stop_reason == "end_turn":
            return resp.content[0].text

        # Add the assistant's tool-use turn
        messages.append({"role": "assistant", "content": resp.content})
        # Run any tool_use blocks, append a tool_result turn
        results = [
            {
                "type": "tool_result",
                "tool_use_id": b.id,
                "content": run_tool(b.name, b.input),
            }
            for b in resp.content if b.type == "tool_use"
        ]
        messages.append({"role": "user", "content": results})
```

Loop until the model says it's done. That's an agent.

---

## What `run_tool` does

```python
def get_weather(city: str) -> str:
    r = httpx.get(f"https://wttr.in/{city}?format=3")
    return r.text

def search_web(query: str) -> str:
    ...

def read_file(path: str) -> str:
    return Path(path).read_text()

TOOLS = {"get_weather": get_weather, "search_web": search_web, "read_file": read_file}

def run_tool(name, arguments):
    return TOOLS[name](**arguments)
```

The model decides **when** and **with what arguments** to call each tool. Your code controls **what they actually do**.

---

## Prompt engineering — the basics

- **Be specific** — "Output a JSON list of titles" beats "make it a list"
- **Give examples** — one or two in the prompt go a long way
- **Set bounds** — "Use at most 100 words" prevents wandering
- **Use system prompts** — for sticky behavior across turns
- **Decompose** — for multi-step tasks, prompt step-by-step or use tools

The skill isn't writing flowery prompts. It's specifying the contract clearly.

---

## RAG — retrieval-augmented generation

When your data isn't in the model's training:

1. **Embed** docs with a vector model
2. **Index** in a vector DB (pgvector, Qdrant, Weaviate, Pinecone)
3. **Retrieve** top-K matching docs for each query
4. **Inject** them into the system prompt
5. **Generate** the answer grounded in retrieved context

```python
docs = vector_db.search(query_embedding, k=5)
system = f"Answer based on these docs:\n\n{format(docs)}"
```

For Q&A over your own corpus, RAG is the default starting point.

---

## Caching, retries, rate limits

```python
import time, functools

def with_retry(fn, retries=5):
    @functools.wraps(fn)
    def go(*a, **kw):
        for i in range(retries):
            try: return fn(*a, **kw)
            except RateLimitError as e:
                time.sleep(2 ** i)
        raise
    return go
```

For expensive prompts, cache by hash of the input. Both Anthropic and OpenAI have prompt-caching APIs to drop the cost of repeated context.

---

## Cost & latency awareness

- **Tokens** = cost. Trim prompts; use small models for orchestration, big models only for the hard step.
- **Streaming** improves perceived latency, not total latency.
- **Tool use** adds round-trips; minimize unnecessary loops.
- **Cache** identical or near-identical prompts.
- **Watch your `max_tokens`** — bigger value, longer wait, higher cost.

---

## Frameworks — when to use one

For one-off integrations, the SDKs above are enough. For larger agentic systems, frameworks like **LangChain**, **LlamaIndex**, **DSPy**, and the official **Anthropic Agent SDK** offer:

- Tool registries, retries, observability
- Vector store / RAG primitives
- Multi-agent orchestration

Reach for them once your hand-rolled loop has 200+ lines and you're fighting the same problems other people have already solved.

---

## Safety basics

- **Never trust the model's output** for security-critical decisions
- **Sanitize tool inputs** — the model is influenced by user input; treat tool args as untrusted
- **Limit tool blast radius** — read-only tools first; gated writes; never `exec()` model output
- **Log prompts and responses** — debugging an agent without logs is impossible
- **Set spending limits** in the provider dashboard. Always.

---

## What's next

- **Async & concurrency** — running many LLM calls in parallel with `asyncio.gather`
- **Deployment** — putting an agent in production
