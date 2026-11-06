# Running a prompt on your other computer without opening a port to everyone on the Wi-Fi

*Development build, October 2026. The first half applies to any local model server; the UCompute half is clearly marked. UCompute is in development. There is no public release or download yet.*

The setup is common: one computer has the memory or the GPU, another is the one you carry. You want a prompt typed on the second to run on the first. Search for how, and the answers fall into three patterns. Each one works. They differ in what they trust to decide who may send prompts, and that is the question worth asking before you pick one.

## Pattern 1: widen the bind, open the firewall

Ollama's FAQ answers "How can I expose Ollama on my network?" with: "Ollama binds 127.0.0.1 port 11434 by default. Change the bind address with the `OLLAMA_HOST` environment variable." Tutorials then set it to `0.0.0.0:11434` and add an inbound firewall rule for the port. LM Studio has a "Serve on Local Network" setting and its docs say: "Any bind other than `127.0.0.1` exposes the server beyond `localhost`", with a recommendation to turn on authentication.

**What it trusts:** the network. Ollama's own authentication page says: "The local API at `http://localhost:11434` does not require authentication." Unless you add a key or a proxy in front, every device that can reach the port can send prompts and read the answers: on a home network, everyone on it; on a dorm, office or guest Wi-Fi, everyone on that. A firewall rule on the private profile narrows which networks; narrowing it to devices takes a per-address rule, and addresses change. (To see what your server is listening on right now, the [sixty-second self-check](2026-11-03-what-0000-exposes.md) has the command for each operating system.)

## Pattern 2: an SSH tunnel

Leave the server on `127.0.0.1` and forward a local port to it over SSH, for example `ssh -L 11434:127.0.0.1:11434 you@other-computer`. OpenSSH describes `-L` as: "Specifies that connections to the given TCP port or Unix socket on the local (client) host are to be forwarded to the given host and port, or Unix socket, on the remote side."

**What it trusts:** an SSH login on the other computer. The model server itself never listens beyond loopback. The costs are that the other computer must run an SSH server you can log in to, and the tunnel lasts as long as the SSH session, so it is something you start rather than something that is just there.

## Pattern 3: a VPN overlay

Put both computers on an overlay network such as Tailscale or WireGuard, and bind the server to the overlay address or serve it through the overlay. Tailscale's own guide (May 2025) runs Ollama and Open WebUI and exposes the stack through Tailscale Serve, "with TLS and no reverse proxy configuration required". (Other guides report that Serve placed directly in front of Ollama needs a Host-header fix.) LM Studio's LM Link is built the same way: "All communication and data transfer between devices is always end-to-end encrypted, thanks to Tailscale."

**What it trusts:** membership of the overlay network, managed through the overlay's accounts and its access rules. This pattern also reaches across networks without opening the model server's port to anyone outside the overlay. The model server behind it often has no authentication of its own unless you turn it on, so the overlay's membership list is the access list; removing a device from the overlay is how you stop it.

## The question all three leave to you

Each pattern answers "can this device reach the port?" None of them, by itself, answers which of your devices may run work here and which may not, whether a given request left this computer, or what to do when the network blocks the discovery a tool relies on. Those are decisions you make with other tools: keys, proxies, overlay rules, logs.

---

## What UCompute does instead (development builds, October 2026)

*This section describes UCompute. UCompute is in development. There is no public release or download yet.*

UCompute Mesh lets you pair your own devices with a short code and choose which one runs a request. Each request runs whole on one device. (Tested with development builds between one Apple M2 Mac and one Windows 11 laptop (Intel Arc 140V) on the same Wi-Fi network or phone hotspot, and between that Mac and one Linux desktop computer (Ubuntu, processor only, a development build of the background program compiled and started by hand; no app or installer) on a home Wi-Fi network. Linux is a planned platform, not a supported one. Pairing between two Macs is not yet shown to work.)

**What it trusts:** a device key pinned at pairing. Traffic between paired devices is encrypted and both ends are authenticated: QUIC with TLS 1.3 and device keys pinned at pairing. Your device will not send work to a device you have not paired. A device you have not paired cannot send you work; that was tested with several instances on one computer, not yet between two machines. The device that runs your request processes your prompt. Pair only devices you control or trust. How the six-digit code becomes a pinned key is in [the pairing note](2026-10-30-pairing-explainer.md).

**Who may run work, request by request.** Your prompts and responses stay on your device unless you pair another device and choose to run a request there. Local-only mode refuses every off-device request. Downloading a model or the engine uses the internet.

**What left the device.** Audit logs record each off-device request without the prompt or the response. ([What the audit log records](2026-10-30-what-the-audit-log-records.md).)

**When discovery is blocked.** Where a network blocks automatic discovery, as some campus Wi-Fi does, paired devices can still connect by address. An address alone is never trusted: the device must present the key it was paired with. Reconnecting by address was shown between one Mac and one Windows laptop on the same campus network, with development builds.

**The Windows firewall step.** On Windows, UCompute asks before it lets your paired devices reach it through the firewall, and the rule it adds covers only its own program, on private networks. That was shown in October 2026 on one Windows 11 laptop with a development build (later builds not yet re-checked); Windows asks for administrator approval.

**Stopping a device.** You can revoke a paired device at any time. In our two-machine tests, its connection closed less than a tenth of a second after the revoke (between 0.002 s and 0.061 s in four runs), and its next request was refused. (Development builds, October 2026; Mac–Windows and Mac–Linux pairs.)

**The model still has to fit.** Before you download a model, UCompute shows its size, its license and whether it fits in your computer's memory. Every download is checked against a pinned SHA-256 hash. That was demonstrated on an Apple M2 Mac (macOS 26.2); the platform is experimental, and memory-fit estimates are calibrated for Apple Silicon Macs. Choosing the other device does not make a bigger model fit: the whole request runs on the device you chose, so the model must fit there.

### The one two-machine run described here

In a test on 19 October 2026 (UTC), development builds of the desktop app on a MacBook Air (Apple M2, macOS 26.2) and a Windows 11 laptop (Intel Core Ultra 7 258V, Arc 140V) were paired through their own windows on one phone hotspot, and the Windows app ran a prompt on the Mac (the Mac's permission for the laptop to run prompts was set through the Mac's local API, not its window). A request in the other direction, sent through the Mac's local API rather than its app window, ran on the laptop's GPU. Each request ran whole on one device; no model was split. The test model was a 15-million-parameter fixture, so speeds from it are not performance figures.

### What this does not do

- It does not work across the internet. The evidence covers devices on the same local network only. If you need to reach a computer from elsewhere, pattern 2 or 3 is still the answer.
- It does not combine two computers' memory. Each request runs whole on one device.
- No platform is declared supported yet. macOS 26.2 on Apple M2 and Windows 11 on an Intel Core Ultra 7 258V with Arc 140V are experimental; Linux, other GPUs and phones (as clients) are planned.

UCompute runs models through a pinned release of llama.cpp (MIT), downloaded on first use and verified against a pinned SHA-256 hash (the digest published for that release); UCompute does not write its own inference kernels. UCompute's code is licensed under Apache-2.0. The source has not been published yet.

---

*Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*

*Follow new notes: [Bluesky @ucompute.org](https://bsky.app/profile/ucompute.org) or the [Atom feed](https://ucompute.github.io/notes/feed.xml).*
