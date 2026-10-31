# What llama.cpp's own README says about its RPC backend, and what UCompute does instead

*Development build, October 2026. UCompute is in development. There is no public release or download yet.*

llama.cpp is the engine UCompute runs models with: UCompute fetches a pinned release of its server on first run and refuses to start it if the archive's SHA-256 does not match the hash pinned in UCompute's catalog. UCompute does not write its own inference kernels. This note is about one specific part of llama.cpp, the RPC backend, because it is the first thing you find when you search for using a second computer's GPU with llama.cpp, and because the llama.cpp project is unusually clear about what it is for.

## What the RPC backend is

The RPC backend lets a llama.cpp process on one host use the accelerator devices of another host over the network (TCP by default, RDMA where both ends support it). You start `ggml-rpc-server` on the remote machine, and on the main machine you pass `--rpc host:port` to `llama-cli` or `llama-server`. Model weights and the KV cache are then split across the local and remote devices, by default in proportion to each device's available memory. This is genuine model partitioning: layers live on the remote device and activations cross the wire during each token.

## What its authors say about it

The RPC README, as of 30 October 2026, says three things that matter:

> "This example and the RPC backend are currently in a proof-of-concept development stage."
>
> "As such, the functionality is fragile and insecure."
>
> "Never run the RPC server on an open network or in a sensitive environment!"

The project's security policy adds, under "Untrusted environments or networks":

> "Do not use the RPC backend, ggml-rpc-server and llama-server functionality […]"

and

> "Encrypt your data if sending it over the network."

Read together, the two documents describe a tool for two processes that trust each other on a network nobody else can reach: encrypt the data yourself, and never expose the port. That is the boundary its authors drew, in their words, and this note takes them at it.

## The advisory the project published

The project's advisory GHSA-j8rj-fmpv-wcxw is dated 26 March 2026, and the CVE record (CVE-2026-34159, CVSS 3.1 base score 9.8, critical) 1 April 2026: in RPC builds up to b7991, a tensor deserialization path skipped bounds validation, and an unauthenticated attacker with TCP access to the RPC server port could read and write the process's memory and, combined with pointer leaks from other RPC messages, run code on the host. The CVE record names b8492 as the fixed build. The project found it, fixed it and documented it. What the advisory shows is that for this component, "whoever can reach the port" is the whole threat model, exactly as the README says. (The release UCompute pins, b11609, is later than that fix, and the Mesh path does not use the RPC backend, as described below.)

## What the warning means for a two-computer setup

The same command that makes the RPC backend work on an isolated lab network also works on a home network, a shared flat, a dorm or an office. A port that accepts tensor uploads and compute commands from any source on the network is what the README is warning about, and a home network, a shared flat, a dorm or an office is not the isolated segment it assumes. Wrapping it in a VPN helps with eavesdropping but still leaves every device inside that VPN able to drive the server.

## What UCompute does instead

UCompute keeps the engine local and puts the network behind a separate, authenticated layer. These are statements about the development build; they become checkable by anyone when the source is published.

- **Engine ports never leave the machine.** UCompute runs llama.cpp's server as a supervised child process bound to loopback, with a random per-process API key. There is no setting to expose that port.
- **The only path between devices is authenticated and encrypted.** Traffic between paired devices is encrypted and both ends are authenticated: QUIC with TLS 1.3 and device keys pinned at pairing. Your device will not send work to a device you have not paired. There is no plaintext peer transport and no downgrade negotiation. A packet capture on one host showed no readable prompt text; a two-host capture has not been done yet.
- **The Mesh path does not use the RPC backend.** UCompute Mesh lets you pair your own devices with a short code and choose which one runs a request. What crosses the network is a request to a paired device, which runs it entirely on its own engine and streams the result back. Each request runs whole on one device; UCompute does not split a model across devices, so a model has to fit the device you choose. (Tested with development builds between one Mac and one Windows laptop, and between that Mac and one Linux computer, on the same local network; the scope is listed under "Where this has run".)
- **So if a model does not fit on either machine alone, UCompute does not solve that today;** the RPC backend does, under the conditions its README sets.
- **The device that runs your request processes your prompt. Pair only devices you control or trust.** Encryption protects the traffic from observers on the network, not from the endpoint.

## Planned, with no date

Running one model across several devices is planned, with no date. It does not exist today. The engineering design that has been written for splitting one model between two devices keeps both ends of the RPC connection on loopback and carries the bytes only inside the same authenticated connection that pairing established, with policy checks on both devices. It is a design under test on one computer, not a feature. This note will be updated when that changes.

## If you want to use the RPC backend anyway

Take the README's warning literally: an isolated network segment that only your two machines can reach, no port forwarding, and an understanding that whoever can reach the port can run arbitrary compute on that machine. The llama.cpp project's own guidance is the right source for its own tool.

## Where this has run

Development builds, October 2026. Pairing and a remote request ran between one Apple M2 Mac (macOS 26.2) and one Windows 11 laptop (Intel Arc 140V) on the same Wi-Fi network or phone hotspot, and between that Mac and one Linux desktop computer (Ubuntu, processor only) on a home Wi-Fi network. On Linux the program was a development build of the background service compiled and started by hand, with no app or installer. In both pairs pairing has been shown starting from either device; with the Linux computer it went through the devices' background programs rather than an app window, and pairing started in the Mac app's window has been shown only with an earlier version of the app. Linux is a planned platform, not a supported one. Pairing between two Macs is not yet shown to work. The transport described above is the design those runs used; the check that captured traffic carries no readable prompt text ran on one computer.

UCompute's code is licensed under Apache-2.0. The source has not been published yet. llama.cpp is MIT-licensed; the quotations above are from its repository and are attributed to its authors.

---

*Published by the UCompute project. This note was drafted and checked by UCompute's automated growth workstream against the project's engineering evidence. Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*
