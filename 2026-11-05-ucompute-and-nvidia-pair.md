# UCompute Mesh and NVIDIA PAIR, fact by fact

*Development build, October 2026. UCompute is in development. There is no public release or download yet. PAIR facts are quoted from its README and SECURITY.md as read on 1 November 2026 and re-checked on 3 November 2026; both documents change, so check them before relying on this page.*

NVIDIA's Personal AI Router (PAIR) and UCompute Mesh answer the same question: you have more than one computer, and you want a prompt typed on one to run on another. Both send each request whole to one machine. Both pair machines with a short code (PAIR's is a six-digit PIN) and then use keys: PAIR's SECURITY.md says its peer traffic is "designed to use certificates and mutual TLS after pairing"; UCompute uses QUIC with TLS 1.3 and pinned device keys. If you are choosing between them today, the first fact decides it: PAIR ships installers, and UCompute has no public release.

This page lines the two up on the facts each project publishes, in each project's own words. It does not rank them.

## The short version

| | NVIDIA PAIR (its README and SECURITY.md) | UCompute Mesh (development build, October 2026) |
|---|---|---|
| Can you install it today? | Yes: "Windows `.exe`; Linux `.deb`; macOS `.dmg`" | No. There is no public release or download yet. |
| License | Apache-2.0 | Apache-2.0. The source has not been published yet. |
| Where a request runs | "PAIR routes each independent request to one node." | Each request runs whole on one device; UCompute does not split a model across devices. |
| Who picks the machine | PAIR: requests "can be routed to eligible nodes according to engine availability, model availability, and current workload" | You choose which device runs a request. |
| Pairing | "The inviting machine shows a six-digit PIN, and you enter that PIN on the machine you invited." | A short code, after which each device pins the other's key. |
| Traffic after pairing | "designed to use certificates and mutual TLS after pairing" | QUIC with TLS 1.3 and device keys pinned at pairing; both ends are authenticated. |
| When a paired machine can run work | "A node only becomes a candidate for a request once it is actually running a compatible engine". The pages read do not describe a separate per-node permission. | Pairing alone does not let another device run work on yours; you also allow it. |
| Record of requests | Each job appears under Jobs, "naming the node that served it"; the pages read do not say what is kept or for how long. SECURITY.md: prompts and response bodies "should not be written to logs". | Audit logs record each off-device request without the prompt or the response. |
| Removing a machine | Any member can remove a node from PAIR's settings | You can revoke a paired device at any time (tested timings below). |
| Engines | "Ollama, LM Studio, and llama.cpp" | A pinned release of llama.cpp, downloaded on first use and verified against a pinned SHA-256 hash. |
| Operating systems | "Windows 11; Linux; macOS", "x64 and arm64 on all three" | No platform is declared supported yet (details below). |

UCompute column: development builds, October 2026, tested between one Mac and one Windows laptop and between that Mac and one Linux computer on the same local network; the full scope is under "Where a request runs". The rest of this page expands each row.

## Availability and platforms

PAIR's README lists "Windows 11; Linux; macOS" with "x64 and arm64 on all three. Windows on ARM is experimental." and installers for each.

UCompute is in development. There is no public release or download yet. No platform is declared supported yet. macOS 26.2 on Apple M2 and Windows 11 on an Intel Core Ultra 7 258V with Arc 140V are experimental; Linux, other GPUs and phones (as clients) are planned. UCompute's code is licensed under Apache-2.0. The source has not been published yet.

That difference matters more than any row below. Everything about UCompute on this page describes development builds you cannot run yet.

## Where a request runs

The two projects make the same promise and the same limit. PAIR's README:

> PAIR routes each independent request to one node. It does **not** pool GPU memory, combine GPUs into a larger logical GPU, shard one model across machines, or split an in-flight inference request between nodes.

UCompute Mesh lets you pair your own devices with a short code and choose which one runs a request. Each request runs whole on one device; UCompute does not split a model across devices. (Tested with development builds between one Apple M2 Mac and one Windows 11 laptop (Intel Arc 140V) on the same Wi-Fi network or phone hotspot, and between that Mac and one Linux desktop computer (Ubuntu, processor only, a development build of the background program compiled and started by hand; no app or installer) on a home Wi-Fi network. Linux is a planned platform, not a supported one. Pairing between two Macs is not yet shown to work.)

The difference is who decides. PAIR is a router: it chooses an eligible machine by engine, model and load, which suits concurrent work such as the "multi-agent applications" its README names. UCompute leaves the choice of device to you. Neither combines two machines' memory, so a model that does not fit on any one of your machines does not fit in either.

## Pairing and the traffic after it

Both use a short code once and keys after that; PAIR's is a six-digit PIN. PAIR's SECURITY.md is direct about what the PIN is for:

> The six-digit PIN is low entropy. Do not treat it as a durable credential or as strong proof of physical presence.

and about the scope of its mutual TLS, which protects PAIR's own channels between paired machines "only when identity and certificate material are present and the relevant workers use that transport".

That is a careful document, and the advice in it (pair only while both devices and the network are trusted) applies to any short-code pairing, UCompute's included.

In UCompute, traffic between paired devices is encrypted and both ends are authenticated: QUIC with TLS 1.3 and device keys pinned at pairing. Your device will not send work to a device you have not paired. Where a network blocks automatic discovery, as some campus Wi-Fi does, paired devices can still connect by address. An address alone is never trusted: the device must present the key it was paired with. (Reconnecting by address was shown between one Mac and one Windows laptop on the same campus network, with development builds.)

## What pairing permits

In PAIR, a paired machine joins the set of nodes that requests can be routed to once it runs a compatible engine and model; the pages read do not describe a further permission step.

In UCompute, pairing and permission are separate: another device can receive work only after you pair it and allow it. (Development builds, October 2026; the group rules were tested with several instances on one computer.) On the sending side, your prompts and responses stay on your device unless you pair another device and choose to run a request there. Local-only mode refuses every off-device request. Downloading a model or the engine uses the internet.

Either way, the device that runs your request processes your prompt. Pair only devices you control or trust.

## A record of what other machines asked for

UCompute's audit logs record each off-device request without the prompt or the response. Each line has a timestamp, an event kind, the other device's id and structured fields such as sizes and durations (development build, October 2026; the mechanics are held by automated tests on one computer); [what the audit log records](2026-10-30-what-the-audit-log-records.md) lists them.

PAIR's README shows each job under Jobs, "naming the node that served it"; the README and SECURITY.md do not say whether that view is kept as a log or what it contains. Its SECURITY.md sets this rule for logs in general: "Prompts, messages, chunks, and response bodies should not be written to logs."

## Removing a machine

PAIR's README gives two ways to take a machine out: "Leave from this machine", or "Remove it from another node", which any member can do from PAIR's settings by removing that node from the list. Its SECURITY.md lists "Pairing, certificates, membership, and member removal" among the areas where changes "require focused review and tests".

In UCompute, you can revoke a paired device at any time. In our two-machine tests, its connection closed less than a tenth of a second after the revoke (between 0.002 s and 0.061 s in four runs), and its next request was refused. (Development build, October 2026; the four runs were between Mac-Windows and Mac-Linux pairs.)

Both projects let you remove a machine. PAIR's README and SECURITY.md do not state a timing; that does not mean it is slow.

## Engines and the local endpoint

PAIR manages "Ollama, LM Studio, and llama.cpp" and presents "Ollama-compatible, OpenAI-compatible, and Anthropic Messages API" endpoints. Its SECURITY.md says a proxy's plaintext side accepts requests "from loopback only".

UCompute runs models through a pinned release of llama.cpp (MIT), downloaded on first use and verified against a pinned SHA-256 hash (the digest published for that release); UCompute does not write its own inference kernels.

## Which one to look at

- If you want to route work across your own machines today, on Windows, Linux or macOS, PAIR is installable and UCompute is not.
- If you want requests spread automatically across machines by load, PAIR is built for that; UCompute leaves the choice of device to you.
- If you want to read how UCompute separates pairing from permission to run work, and what its audit log records, see [how two devices pair](2026-10-30-pairing-explainer.md); that describes development builds, with no release yet.

Corrections from either project's users or maintainers are welcome at hello@ucompute.org.

---

*Published by the UCompute project. This note was drafted and checked by UCompute's automated growth workstream against the project's engineering evidence. Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*

*Follow new notes: [Bluesky @ucompute.org](https://bsky.app/profile/ucompute.org) or the [Atom feed](https://ucompute.github.io/notes/feed.xml).*
