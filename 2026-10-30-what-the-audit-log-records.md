# What UCompute's audit log records, and what it never records

*Development build, October 2026. UCompute is in development. There is no public release or download yet. The mechanics below are held by automated tests on one computer; the absence of prompt text in the logs was checked in the two-machine runs listed at the end.*

If you let a second device on your local network run your prompts, two questions follow: what did I send where, and did any of it end up in a log? UCompute answers the first with an audit trail and the second with rules the audit code enforces on itself. Audit logs record each off-device request without the prompt or the response.

## What is recorded

The audit trail lives in the data directory as `logs/audit.jsonl`: one JSON object per line with a UTC timestamp, an event kind, the other device's id where one is involved, and structured fields: ids, sizes, reasons, durations. On macOS, and on the Linux development build (Linux is a planned platform, not a supported one), the file is created readable only by the user account that owns it; on Windows it carries the access rights of the user's profile folder, and access from a second Windows account has not been tested. A sent request looks like this (shape, not a real record):

```json
{"ts_utc":"2026-10-30T12:00:00Z","kind":"remote_request","peer_id":"<device id>",
 "fields":{"phase":"finished","request_id":"<id>","outcome":"ok","completion_tokens":64,"duration_ms":1830}}
```

The kinds that exist in development builds:

| Event | When it is written |
|---|---|
| `pairing_started`, `pairing_succeeded`, `pairing_failed`, `pairing_locked` | a pairing window opens, completes, fails, or locks after repeated failures |
| `peer_connected`, `peer_disconnected` | a paired device connects; a connection ends |
| `peer_rejected` | a connection or request is refused: an unpaired or revoked device, a device you disconnected, serving turned off, or a group that does not allow remote work |
| `remote_request` | the sending side writes a line when a request leaves (`phase: sent`) and one when it ends (`phase: finished`, with the outcome and the request id); the serving side writes one line when the request ends, with the same request id, token counts and outcome |
| `peer_revoked` | you revoke a device, or a revocation from another member of the group is applied |
| `peer_removed` | you unpair a device |
| `privacy_changed`, `network_changed`, `local_priority_changed`, `group_updated` | you change a setting that governs what may leave or enter the device |
| `audit_rotated` | the file rolled over; the line states how many bytes were rotated into the kept files and how many were dropped |

Two properties of the `remote_request` lines matter most:

- **The `sent` line is written before the request leaves the device.** If the line cannot be written, nothing is sent. So there is no path by which a request reaches another device without a record on yours.
- **Both devices carry the same request id.** When you run a request on your desktop from your laptop, your laptop has the `sent` and `finished` lines and the desktop has its serving line, all with one id.

A request that policy stops on the sending side writes no line at all: nothing left the device.

You read the trail with `ucompute audit` or `GET /api/audit` on the local API; the diagnostics bundle includes it.

## What is never recorded

The audit code builds its fields from ids, sizes, reasons and durations and never receives prompt text. Two backstops guard against a mistake: the writer refuses any field named `prompt`, `content`, `messages`, `api_key`, `private_key` or `pkcs8`, matched case-insensitively at any depth, and any string value longer than 2 KiB. The device's private key is never formatted into any log, and every string in the diagnostics bundle a user can export passes through redaction before it is written.

In the two-machine runs that checked it, the audit logs on both devices contained no prompt or response text. The project's stated rule goes further, no prompt in engine logs, API logs, metrics, error messages or run records, but today only the audit log and the diagnostics bundle are held by tests; the rest is a design rule with its tests still to come.

## Why the trail is bounded, and why a peer flood cannot rotate away owner actions and revocations

`audit.jsonl` rotates at 8 MiB into `audit.1.jsonl` through `audit.3.jsonl`, so a trail is bounded at roughly 32 MiB. An unbounded log is a disk-filling attack; the cost of the bound is that long history is dropped, and the design makes the drop visible: every new file begins with an `audit_rotated` line that states the dropped bytes.

A paired device could try to use that rotation against you: flood you with requests, and the lines you care about, a revocation or a privacy change, would scroll out. So the kinds a remote device cannot produce at will, the owner actions and the revocations, are also copied to `audit.security.jsonl`, rotated on its own. Lines a peer can trigger repeatedly, such as refused requests or connections, are not copied there, or the flood would rotate that file away too. Refused-connection lines are additionally capped at 20 per minute in the main trail; beyond that they are counted, and the count rides on the next line written. On disk the two trails stay separate; the product's reader adds the security copies back in when the main trail has rotated them away.

## What this does not do

- It does not hide your prompt from the device that runs it. The device that runs your request processes your prompt. Pair only devices you control or trust. The audit trail tells you that a request went to that device; it cannot tell you what the device did with it.
- It does not record local-only work. Your prompts and responses stay on your device unless you pair another device and choose to run a request there. Local-only mode refuses every off-device request. Downloading a model or the engine uses the internet.
- It is not a forensic archive. What is dropped at rotation is reported, not recovered.
- There is one serving device per request. Each request runs whole on one device (tested with development builds between one Mac and one Windows laptop and between that Mac and one Linux computer; the full scope is below).

## Where this has run

Development builds, October 2026. Pairing and remote requests ran between one Apple M2 Mac (macOS 26.2) and one Windows 11 laptop (Intel Arc 140V) on the same Wi-Fi network or phone hotspot, and between that Mac and one Linux desktop computer (Ubuntu, processor only, a development build of the background program compiled and started by hand; no app or installer) on a home Wi-Fi network. In both pairs pairing has been shown starting from either device; with the Linux computer it went through the devices' background programs rather than an app window, and pairing started in the Mac app's window has been shown only with an earlier version of the app. Linux is a planned platform, not a supported one. Pairing between two Macs is not yet shown to work. The audit logs of both devices were scanned for prompt or response text after those runs and held none. The refusal of content-bearing fields, the rotation behaviour, the security trail surviving a request flood, and the redaction of the diagnostics bundle are held by automated tests on one computer.

UCompute runs models through a pinned release of llama.cpp (MIT; b11609 in the October 2026 development builds), downloaded on first use and verified against a pinned SHA-256 hash; UCompute does not write its own inference kernels. UCompute's code is licensed under Apache-2.0. The source has not been published yet.

---

*Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*

*Follow new notes: [Bluesky @ucompute.org](https://bsky.app/profile/ucompute.org) or the [Atom feed](https://ucompute.github.io/notes/feed.xml).*
