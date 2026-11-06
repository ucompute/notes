# A shared API key is not device trust

Let a second computer use the model server on your desktop, and the server has to decide, for every request, whether to answer. Five local AI servers make that decision in four different ways. Their own words are quoted below, as read on 5 November 2026 (exo's on 30 October). Docs change, so check before relying on them.

Ollama, llama-server, Jan and LM Studio all listen only on your own computer by default. That's the safe place to start, and everything here is about what happens when you leave it.

## The four answers

| What the server checks | Project and its own words |
|---|---|
| **Nothing; add your own in front** | Ollama: "The local API at `http://localhost:11434` does not require authentication." and "Ollama binds 127.0.0.1 port 11434 by default." and "Ollama runs an HTTP server and can be exposed using a proxy server such as Nginx." exo, in a reply from a project member closing an issue about unauthenticated endpoints (read 30 October 2026): "this is intended behaviour, exo's api is unauthorized" and "run exo behind a reverse proxy and provide your own authentication if you wish to expose it downstream." |
| **One shared key** | Jan: "If a key is configured, requests must include it in the `Authorization: Bearer YOUR_API_KEY` header." and "Jan refuses to start the server on `0.0.0.0` (or any address other than your own computer) without a key." |
| **A list of keys** | llama.cpp `llama-server`: `--api-key` is the "API key to use for authentication, multiple keys can be provided as a comma-separated list (default: none)"; `--api-key-file` reads "API keys, one per line". `/health` "is public (no API key check)". |
| **Named tokens with permissions** | LM Studio: "By default, LM Studio does not require authentication for API requests." When you turn it on, you create tokens: "Provide a name for the token and select the desired permissions." Its page on serving to the network says: "Any bind other than `127.0.0.1` exposes the server beyond `localhost`; we recommend enabling authentication." |

Ollama, llama-server, Jan and LM Studio each document `127.0.0.1` as the default; Jan and LM Studio also warn against leaving it without a key or token. The rows differ in what happens once you bind to something else.

## What a key tells the server, and what it does not

A bearer key answers one question: does this request carry a secret the server knows? That is worth having. It stops the casual case, a neighbour's laptop or a scanner that finds the port.

It does not answer other questions you may care about once more than one device uses the server:

- **Which device sent this?** With one shared key, every device that holds it looks the same to the server. If the key is in your laptop's shell profile, in a notebook on a lab machine and in a phone app, the server cannot tell those apart.
- **How do I stop one device?** With one shared key, you change the key and then give the new one to every device you still want. With a list of keys (llama-server) or named tokens (LM Studio), you can give each device its own and remove one (LM Studio's token list lets you delete a token; with llama-server you edit the list or key file and restart the server); that is the reason to prefer them, and to use one key per device rather than one key for all.
- **Who could read the key on the way?** The examples on these pages use `http://` URLs: `http://127.0.0.1:1337` (Jan), `http://localhost:1234` (LM Studio), `http://127.0.0.1:9931` (llama-server). Over plain HTTP, the key and the prompt cross the network as readable text to anything positioned to see the traffic. On loopback that path never leaves your computer; on Wi-Fi it does. llama-server documents `--ssl-key-file` and `--ssl-cert-file` and a build "with SSL support using OpenSSL 3"; for the others, check the current documentation, or keep the server on loopback and reach it through an SSH tunnel or a VPN overlay ([how those work](2026-11-04-other-computer-without-open-port.md)).
- **What was asked from where?** A key check accepts or refuses. Jan and llama-server document request and prompt logging for debugging; a per-device record of requests that leaves the prompt out is not something the pages quoted above describe.

None of this makes keys a bad idea. If you serve a model to other devices today, use a key, one per device where the server allows it, keep `/health`-style public endpoints in mind, and prefer a tunnel or overlay to an open port.

## A five-minute checklist for a server you already expose

1. Find out what it is listening on: the [sixty-second self-check](2026-11-03-what-0000-exposes.md) has the command for each operating system.
2. If it is not on `127.0.0.1`, turn on its key or token setting, using the wording on its own documentation page; Ollama and exo document none for their local API, so put a reverse proxy with authentication in front, as exo's reply suggests.
3. Give each device its own key where the server supports more than one (llama-server's comma-separated list or key file; LM Studio's named tokens).
4. Write down which key went where, so you can remove one without rotating all of them.
5. If the traffic crosses Wi-Fi as plain HTTP, move it into a tunnel or an overlay.

---

## What UCompute does about this (development build, October 2026)

UCompute is in development. There is no public release or download yet.

Instead of a key, devices pair. You pair your own devices with a short code and choose which one runs a request, and each request runs whole on one device. (Tested with development builds between one Apple M2 Mac and one Windows 11 laptop (Intel Arc 140V) on the same Wi-Fi network or phone hotspot, and between that Mac and one Linux desktop computer (Ubuntu, processor only, a development build of the background program compiled and started by hand; no app or installer) on a home Wi-Fi network. Linux is a planned platform, not a supported one. Pairing between two Macs is not yet shown to work.)

After pairing, traffic is encrypted and both ends are authenticated: QUIC with TLS 1.3, with device keys pinned at pairing. Your device will not send work to a device you have not paired. The device that runs your request still processes your prompt, so pair only devices you control or trust.

You can revoke a paired device at any time. In our two-machine tests (development builds, October 2026; a Mac paired with a Windows laptop and with a Linux desktop computer), its connection closed less than a tenth of a second after the revoke. Audit logs record each off-device request without the prompt or the response ([what the audit log records](2026-10-30-what-the-audit-log-records.md)).

### What this does not do

- It does not replace a key on a server you already run. UCompute's pairing applies only between UCompute devices; an Ollama, exo, llama-server, Jan or LM Studio server you expose keeps whatever check it has.
- It has not been shown across the internet. The evidence covers devices on the same local network only.
- No platform is declared supported yet. macOS 26.2 on Apple M2 and Windows 11 on an Intel Core Ultra 7 258V with Arc 140V are experimental; Linux, other GPUs and phones (as clients) are planned.

---

*Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*

*Follow new notes: [Bluesky @ucompute.org](https://bsky.app/profile/ucompute.org) or the [Atom feed](https://ucompute.github.io/notes/feed.xml).*
