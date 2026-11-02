# UCompute engineering notes

Technical notes from the UCompute project: how its development builds work, what has been tested and where, and what does not exist yet. UCompute is in development. There is no public release or download yet. UCompute's code is licensed under Apache-2.0. The source has not been published yet.

Not related to the uCompute RP2040 boards, Universal Compute products or any token.

- [How two UCompute devices pair, allow, audit and revoke: the six-digit code and the pinned keys](2026-10-30-pairing-explainer.md)
- [What UCompute's audit log records, and what it never records](2026-10-30-what-the-audit-log-records.md)
- [What llama.cpp's own README says about its RPC backend, and what UCompute does instead](2026-10-31-loopback-rpc-vs-authenticated-transport.md)
- [How to check a downloaded GGUF against its published SHA-256, and what UCompute's development builds check](2026-10-31-verify-a-gguf-download.md)
- [Before you load a model: how UCompute estimates whether a GGUF fits](2026-11-01-memory-fit-explainer.md)
- [Point the official OpenAI Python and JavaScript clients at a local server](2026-11-02-openai-client-base-url.md)
- [What binding a local AI server to 0.0.0.0 exposes, and a sixty-second self-check](2026-11-03-what-0000-exposes.md)
- [Running a prompt on your other computer without opening a port to everyone on the Wi-Fi](2026-11-04-other-computer-without-open-port.md)

Feed: [feed.xml](feed.xml). Each note states the hardware and builds it rests on; numbers are never performance claims.
