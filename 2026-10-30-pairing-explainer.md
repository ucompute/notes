# How two UCompute devices pair, allow, audit and revoke: the six-digit code and the pinned keys

*Development build, October 2026. UCompute is in development. There is no public release or download yet. This note describes behaviour as built in development builds. The protocol mechanics (the pairing confirmation, the relay check, the lockout, no session resumption, the revocation rules) are verified by automated tests on one computer; pairing, remote execution, revocation and connect-by-address have also run between the two-machine pairs listed at the end.*

You have a laptop and a second computer. You want a prompt typed on the laptop to run on the other machine. The usual way to do this with a local model server is to bind it to every network interface and point the laptop at the other machine's address. That works, and it also means the port is the whole trust boundary: any client on the network that can reach it can send prompts and read answers. UCompute Mesh takes a different route: two devices pair once with a short code; a paired device still refuses every off-device request until you allow it; each off-device request it does run is recorded without the prompt or the response; and you can revoke a device at any time.

UCompute Mesh lets you pair your own devices with a short code and choose which one runs a request. Each request runs whole on one device; UCompute does not split a model across devices. (Tested with development builds between one Mac and one Windows laptop, and between that Mac and one Linux computer, all on the same local network; the exact scope is listed under "Where this has run".)

## Every device has one key

Each UCompute device holds one Ed25519 key pair, generated on the device and kept across restarts. That key is the device's identity. There are no certificates, no certificate authority and no accounts: the peer transport is QUIC with TLS 1.3 using raw public keys (RFC 7250), both ways. When two devices connect, each one proves it holds the private key for a public key the other has already pinned. If the key is not pinned, the handshake fails before any application data moves.

So pairing is the act of getting the other device's public key into your pin list in a way you can trust, plus learning its name and service port. It grants no permission to run work; the policy layer below decides that.

## Pairing: a six-digit code, used once

Typing a long public-key fingerprint on a second machine is error-prone, so pairing uses a short code and a password-authenticated key exchange (SPAKE2). The order of events, as built:

```
  Device A (shows the code)                 Device B (types the code)
  ─────────────────────────                 ─────────────────────────
  opens a pairing endpoint,
  120-second window, 6-digit code  ──────▶  user types the code
                                   ◀──────  TLS 1.3 connection (any key accepted here;
                                            trust comes from the code, not the handshake)
                                   ◀──────  SPAKE2 message B
  SPAKE2 message A                 ──────▶
  both derive the shared secret K
  both read a 32-byte value exported from this TLS session
  MAC_A = HMAC(K, "A" ‖ exporter ‖ idA ‖ idB)  ──────▶
                                   ◀──────  MAC_B = HMAC(K, "B" ‖ exporter ‖ idA ‖ idB)
  each checks the other's MAC in constant time
  only now: exchange device name and service port
  both pin the other's key; both write an audit line
```

Three details carry the security. The relay and lockout checks ran in automated tests on one computer.

1. **The confirmation is bound to this TLS session and to both identities.** The MAC covers a value exported from the TLS session and both device ids. A relay that terminates TLS on each side presents its own key on each leg, so the exported values differ, both MACs fail, pairing fails on both ends and nothing is pinned. A wrong code fails the same check, so the log cannot tell a typo from a relay; it records a failed confirmation either way.
2. **Name and port are withheld until the MAC verifies.** The device name and the service port are sent only after the confirmation succeeds. A party that does not know the code learns neither. (It can see that a device is in pairing mode, and that device's public id, because the pairing window is advertised on the local network while it is open.)
3. **Guessing is bounded.** The code is six uniformly random digits, single use, valid for 120 seconds. The device handles one attempt at a time, each with a ten-second limit, and after the sixth failure within 60 seconds the pairing endpoint closes for that window; the next attempt needs a new code. That is at most six guesses against a one-in-a-million code.

A QR variant carries the device's public key, its address and a one-time secret that takes the place of the typed code as the SPAKE2 password. A device that scanned the QR accepts only that key inside the handshake; if another device answers at that address, nothing is pinned.

## After pairing: the handshake is the gate

Once both devices hold each other's keys, every connection between them is a mutually authenticated TLS 1.3 handshake. Some choices that follow from "the verdict is inside the handshake":

- **No session resumption.** TLS 1.3 resumption would skip the client-key check on reconnect, which is exactly where a revoked device would try to return. UCompute issues no tickets and keeps no session cache, so every connection is a full handshake against the current pin list.
- **No 0-RTT, no plaintext path.** There is no unencrypted peer transport and no downgrade negotiation; TLS 1.3 only.
- **Address validation before cryptography.** The peer endpoint used after pairing answers an unvalidated source with a QUIC Retry first, so spoofed packets never reach the TLS handshake. (The pairing endpoint instead relies on its one-at-a-time loop and lockout.)
- **A refusal is only believed from the device itself.** For a short time after the handshake, QUIC still accepts packets of the Initial type, whose keys derive from public information, so a close carried that way can be forged on the path. UCompute therefore treats any close or alert that arrives before the dialler has read the other device's own hello on the control stream as unproven: the dialler reports that the device at that address did not prove its identity, and tries the next address. Every address is a hint; the key is the identity. This is also why paired devices can reconnect by address on networks that block discovery (shown between one Mac and one Windows laptop on the same campus network, with development builds): an address alone is never trusted.
- **Your device will not send work to a device you have not paired.**

Traffic between paired devices is encrypted and both ends are authenticated: QUIC with TLS 1.3 and device keys pinned at pairing. A packet capture that checked for readable prompt text ran on one computer; a capture between two machines has not been done yet.

## Authentication is not authorization

A pinned key proves which device is on the other end. It does not decide whether that device may do anything. That is a separate policy layer:

- **Networking is off by default**, and nothing listens beyond loopback until you turn it on.
- **Local-only is the default privacy mode.** Your prompts and responses stay on your device unless you pair another device and choose to run a request there. Local-only mode refuses every off-device request. Downloading a model or the engine uses the internet; the local-only rule covers prompts and responses. The policy check runs before any network activity, again before a stream opens, and again on the serving side.
- **Leaving local-only requires an acknowledgment.** The disclosure states, before you can choose a mode that sends work elsewhere, that the device running your request reads it; the API requires an explicit acknowledgment flag, the command line asks for a yes.
- **Serving is opt-in and per group.** A paired device may send you work only if you turned serving on and the two devices share a group that allows it. Per-device buckets bound requests and tokens per minute, and the serving side reserves one engine slot for local work. This reserves capacity; whether your own work stays unslowed under a peer flood is not yet verified.

Audit logs record each off-device request without the prompt or the response. The audit line for a sent request is written before the request leaves the device; if it cannot be written, nothing is sent.

## Revocation: a signed record, and what the two-machine runs measured

You can revoke a paired device at any time. A revocation is a signed record naming the target device, with a per-issuer sequence number. A receiving device accepts it only if the issuer is itself pinned there and not revoked, the signature verifies under the issuer's key, and the issuer and the target share a group there, so a stranger cannot cut off your devices. When a device re-pairs with a target it had revoked, it marks its own earlier records as superseded, so a replay of a revocation it has already seen cannot undo the re-pair (a record it never saw can still re-revoke; that is a known limit, and it fails closed). Every open connection to the revoked device closes at once, its key is refused inside the next handshake, and because there is no session resumption, there is no cached ticket to come back with. Other devices in the shared group receive the record over their existing connections and again on every reconnect until they confirm it.

In our two-machine tests, its connection closed less than a tenth of a second after the revoke (between 0.002 s and 0.061 s in four runs), and its next request was refused. Three of those runs were between the Mac and the Windows laptop, one between the Mac and the Linux computer.

## What this does not do

- It does not hide your prompt from the device that runs it. Encryption protects the traffic from observers on the network, not from the endpoint. The device that runs your request processes your prompt. Pair only devices you control or trust.
- It does not pool devices. Each request runs whole on one device; UCompute does not split a model across devices. Running one model across several devices is planned, with no date.
- It does not work across the internet. The evidence covers two devices on the same local network. There is no relay and no NAT traversal.
- It is not a stable protocol. One implementation exists and its peer profile is experimental.

## Where this has run

Development builds, October 2026:

- **Pairing and a remote request:** between one Apple M2 Mac (macOS 26.2) and one Windows 11 laptop (Intel Arc 140V), on the same Wi-Fi network or phone hotspot; and between that Mac and one Linux desktop computer (Ubuntu, processor only) on a home Wi-Fi network. On Linux the program was a development build of the background service compiled and started by hand, with no app or installer.
- **Pairing started from either device:** shown in both pairs. With the Linux computer it went through the devices' background programs rather than an app window. Pairing started in the Mac app's own window has been shown only with an earlier version of the app.
- **Reconnecting by address** where a network blocks automatic discovery, as some campus Wi-Fi does: shown between the Mac and the Windows laptop on the same campus network. An address alone is never trusted: the device must present the key it was paired with.
- **Revocation timing:** four two-machine runs (three Mac–Windows, one Mac–Linux).
- **Captured traffic carries no readable prompt text:** checked on one computer.
- **A device you have not paired cannot send you work:** tested with several instances on one computer, not yet between two machines.
- **Two Macs:** pairing between two Macs is not yet shown to work.
- **Linux** is a planned platform, not a supported one.

No platform is declared supported yet. macOS 26.2 on Apple M2 and Windows 11 on an Intel Core Ultra 7 258V with Arc 140V are experimental; Linux, other GPUs and phones (as clients) are planned.

UCompute runs models through a pinned release of llama.cpp (MIT; b11609 in the October 2026 development builds), downloaded on first use and verified against a pinned SHA-256 hash; UCompute does not write its own inference kernels. UCompute's code is licensed under Apache-2.0. The source has not been published yet. When it is, this note will link to the modules and the tests that hold each rule above in place.

---

*Published by the UCompute project. This note was drafted and checked by UCompute's automated growth workstream against the project's engineering evidence. Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*
