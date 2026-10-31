# Before you load a model: how UCompute estimates whether a GGUF fits

*Development build, October 2026. UCompute is in development. There is no public release or download yet.*

A common first failure with a local model is not a crash. It is a machine that starts swapping and a first token that arrives a minute late, because the model file was about the size of the free memory and everything the engine allocates on top of the weights pushed it over.

Before you download a model, UCompute shows its size, its license and whether it fits in your computer's memory. Every download is checked against a pinned SHA-256 hash. *Demonstrated on an Apple M2 Mac (macOS 26.2); the platform is experimental. Memory-fit estimates are calibrated for Apple Silicon Macs.* This note explains the "fits" part: what is known before a download and what is known after, what the full estimate counts, where its constants come from, and how far off it has been measured to be.

## Two checks, not one

**Before the download**, UCompute knows only what the catalog records about a file: its size, its license, its hash, its family and quantization. None of that says how many layers the model has or how big its vocabulary is. So the pre-download verdict is a lower bound: the file size plus the engine's fixed process baseline, with no KV cache and no compute buffers. "Fits" at that stage means "the weights alone fit". A model that fails even that bound is not worth the bandwidth.

**After the download**, the file's header is on disk, and the full estimate below runs against the memory available at that moment, at the model's trained context length capped at 4,096 tokens, keeping ten percent of available memory spare. It runs again before every load, because a browser with forty tabs changes the answer. The catalog view says which of the two estimates it is showing.

## The file size is not the footprint

A GGUF file is mostly weights, and the weights are one of four things the engine keeps in memory:

| Component | What it is | How UCompute estimates it |
|---|---|---|
| **Weights** | The tensors in the file, minus the header | File size minus the data offset. If the model ties its input and output embeddings (no separate output tensor), the engine keeps a second copy of the token embedding on the compute device, so that tensor is counted twice |
| **KV cache** | The attention keys and values for every token in the context | Per-token bytes from the model's layer count, KV heads and head size, times the context length, with the context rounded up to whole blocks of 256 cells because that is how the engine allocates it (asking for a 128-token context allocates 256 cells) |
| **Compute buffers** | Scratch space for one batch of tokens through the graph | Batch size (512 by default, capped at the context) times a per-token cost that scales with the feed-forward width, plus a per-cell cost for the attention mask |
| **Process baseline and tokenizer tables** | The engine process itself, its libraries, the GPU device, and the vocabulary maps | A measured fixed baseline (46 MiB on macOS), plus a per-token cost for the tokenizer. Byte-pair tokenizers keep token and merge maps that measured about 1.1 KiB per vocabulary entry on macOS; Qwen2.5's 151,936-entry vocabulary is therefore about 160 MiB before a single layer loads, more than that model's own KV cache at a 4,096-token context. We have not isolated why the per-entry cost is so much lower on Linux (see below) |

If the total does not fit within the budget, the verdict is a plain sentence. The tested output reads:

> This model needs about 5.2 GB but only 3.1 GB is available.

## Where the constants come from

The macOS coefficients were measured against the pinned engine release on an Apple M2 Mac, by reading the engine's own log lines for the model buffer, the KV cache and the compute buffer, and the process's peak physical footprint with a filled context. The constants live in one place in the code, each with the measurement that produced it, so a new engine release means re-measuring, not guessing.

Measured on that Mac (macOS 26.2, development build) across fourteen model, context, KV-type and device combinations, from the 260,000-parameter stories260K test model to Qwen2.5 1.5B Instruct (Q4_K_M), the estimate landed between 4.6 % below and 8.9 % above the measured peak physical footprint. Weights and KV bytes matched the engine's log exactly. Four of the rows, estimate / measured in MiB:

| Model | Context | Device | Estimate | Measured |
|---|---|---|---|---|
| qwen2.5-0.5b-instruct q4_k_m | 4,096 | Metal | 767 | 769 |
| smollm2-360m-instruct q8_0 | 8,192 | Metal | 871 | 876 |
| qwen2.5-1.5b-instruct q4_k_m | 4,096 | Metal | 1,464 | 1,492 |
| stories260K (test model) | 128 | CPU | 48 | 44 |

**Nothing above 1.5 billion parameters has been measured yet.** The KV and compute terms are linear in the model's dimensions, so they should extrapolate, but the baseline and the headroom have not been checked at larger sizes. Treat the estimate for larger models as an estimate.

One Linux desktop computer (processor only; Linux is a planned platform, and this is a development measurement on one machine) measured a much smaller tokenizer and baseline overhead, about 390 bytes per vocabulary entry and a 20 MiB baseline; the development build carries those figures for Linux, but they are outside the calibrated claim, which covers Apple Silicon Macs. Windows is not yet separately measured and uses the macOS figures as a placeholder; which direction that errs on Windows has not been checked.

## What this does not claim

- It is an estimate with deliberate headroom, not a guarantee.
- It has been calibrated on one engine release and on models up to 1.5 billion parameters.
- No platform is declared supported yet. macOS 26.2 on Apple M2 and Windows 11 on an Intel Core Ultra 7 258V with Arc 140V are experimental; Linux, other GPUs and phones (as clients) are planned.

UCompute runs models through a pinned release of llama.cpp (MIT; b11609 in the October 2026 development builds, the release the constants above were measured against), downloaded on first use and verified against a pinned SHA-256 hash; UCompute does not write its own inference kernels. UCompute's code is licensed under Apache-2.0. The source has not been published yet.

---

*Published by the UCompute project. This note was drafted and checked by UCompute's automated growth workstream against the project's engineering evidence. Please do not post suspected security problems publicly; a private reporting route will be listed here when it exists. UCompute is in development. There is no public release or download yet.*
