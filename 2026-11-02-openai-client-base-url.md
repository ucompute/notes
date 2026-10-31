# Point the official OpenAI Python and JavaScript clients at a local server

*Development build, October 2026. UCompute is in development. There is no public release or download yet. The client-side steps below apply to any server that speaks the OpenAI chat-completions API; the UCompute-specific details are in their own section.*

If your application already uses the official OpenAI client libraries, you do not need a different SDK to talk to a model running on your own machine. Both libraries take a base URL. Change it and the model id, and keep the rest of your code.

## Python

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:8192/v1",   # your local server
    api_key="unused",                      # the library requires a value; a local server may ignore it
)

stream = client.chat.completions.create(
    model="qwen2.5-0.5b-instruct-q4_k_m",  # example id; use one from GET /v1/models on your server
    messages=[{"role": "user", "content": "Explain unified memory in two sentences."}],
    stream=True,
)
for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```

## JavaScript / TypeScript

```ts
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "http://127.0.0.1:8192/v1",
  apiKey: "unused",
});

const stream = await client.chat.completions.create({
  model: "qwen2.5-0.5b-instruct-q4_k_m", // example id; use one from GET /v1/models on your server
  messages: [{ role: "user", content: "Explain unified memory in two sentences." }],
  stream: true,
});
for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
}
```

## Three things that trip people up

1. **The `/v1` suffix.** Most local servers mount the OpenAI routes under `/v1`. If you get a 404 on `chat/completions`, that is usually why.
2. **The API key.** The libraries refuse to construct a client without one, even when the server ignores it. Pass any non-empty string, or set `OPENAI_API_KEY` in the environment to a placeholder. Never put a real cloud key in a local configuration you will commit.
3. **The model id.** A local server exposes its own ids. List them with `GET /v1/models` on your base URL, and use one of those strings; the cloud model names will not exist there.

## What UCompute does (development build)

UCompute's local API works with the official OpenAI Python and JavaScript client libraries; only the base URL changes. Demonstrated on an Apple M2 Mac (macOS 26.2); the platform is experimental. Chat and completions with streaming were exercised; tool calling is not yet verified. The model id in the examples is the id of one model in UCompute's catalog, the one its own quick start uses.

- `ucompute serve` starts the server on `127.0.0.1:8192`. By default only programs on the same computer can reach the HTTP API; it does not listen on your network address unless you configure it to. To use a second device, pair it instead; how pairing works is covered in a separate note.
- UCompute runs small open models on your own computer, with no account and no UCompute service. By default, your requests stay on your device: your prompts and responses stay on your device unless you pair another device and choose to run a request there, and local-only mode refuses every off-device request. Downloading a model or the engine uses the internet. The device that runs your request processes your prompt. Pair only devices you control or trust. How pairing works is covered in a separate note. Demonstrated on an Apple M2 Mac (macOS 26.2) and a Windows 11 laptop with an Intel Arc 140V GPU; both platforms are experimental. No platform is declared supported yet; Linux, other GPUs and phones (as clients) are planned.
- The API also serves an OpenAPI 3.1 document at `/openapi.json`, which lists the routes and the request fields each one reads, and Prometheus metrics at `/metrics`, designed to carry no prompt or completion text (a design rule; today the audit log and the diagnostics bundle are the parts held by tests).

## What actually changes

The same SDK and the same streaming loop; mostly the same error handling, though a local server returns its own error bodies and has no rate-limit headers. What changes is where the request goes: by default, to a server on your own computer. For development and tests, the change is one line of client configuration.

UCompute runs models through a pinned release of llama.cpp (MIT; b11609 in the October 2026 development builds), downloaded on first use and verified against a pinned SHA-256 hash; UCompute does not write its own inference kernels. UCompute's code is licensed under Apache-2.0. The source has not been published yet.

---

*Published by the UCompute project. This note was drafted and checked by UCompute's automated growth workstream against the project's engineering evidence. Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*
