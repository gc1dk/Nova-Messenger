# Nova Messenger — Plan

> **!coming soon!**
>
> Nova is in active development. Star the repo to follow along.

Nova is a private messenger with no servers and **no internet**. It never
requests mobile data, never phones home, and never asks for an account — there
is nothing to sign up to. Everything travels directly between phones over the
Wi-Fi you are standing on, a hotspot you make together, Wi-Fi Direct, or a
chain of paired devices that carries a message past where the network ends.

That is not a reduced feature set working offline. That is the design.

---

## How it will work

### Zero internet, zero mobile data

Nova uses one permission any socket app technically needs on Android
(`INTERNET`), and uses it for nothing but local paths: the LAN you are on, the
hotspot between two phones, a device-to-device link, or relaying through
friends. Airplane mode with Wi-Fi on is a normal way to run Nova. Turn both
off and Nova simply has no paths — it queues, and sends the moment there is
one.

### Pairing, not accounts

Two devices pair by exchanging public keys — in person, over a QR code. That
exchange **is** the account system. There is no registry of users to look you
up in, because there is no registry.

### Finding each other on the network

On shared Wi-Fi — even a router with no internet connection — devices announce
themselves with a small UDP beacon and listen for others. Each side verifies a
neighbour's self-signed keys before believing anything, so a stranger on the
café's Wi-Fi cannot inject an identity. No rendezvous server, so no server to
block, log, or subpoena — and nothing to keep running.

### Talking without a server

Every message is encrypted with a double ratchet (the same pattern Signal
uses): a session key that advances with every message, so a compromised key
does not open past or future messages, and deleted really means deleted. The
initial key agreement is X3DH over prekeys, so one side can start a
conversation while the other is offline — the message waits for them to come
back.

### Across rooms, floors, and networks — the mesh

A peer that is not on this network is not out of reach. When a direct path
does not exist, the same encrypted frame is handed to any other paired device
that is present, and it is carried onward — through a neighbour's phone in
between, up a floor, over to another hotspot — until it reaches you.

The logic, and the reason this is safe to allow:

- A message is **sealed end to end** between the two endpoints before the
  first hop. Every device in between forwards it blind: it cannot read the
  message, and the recipient cannot tell which path it travelled.
- Each hop wraps the frame in **its own transport encryption with its own
  neighbour**, so no stranger can inject or redirect traffic mid-chain.
- A hop limit and a per-frame random id **stop loops and duplicate floods**:
  a copy that comes back around a chain is dropped the first time it is seen.
- Hops are bounds-checked and the dedup window is capped, so a malicious
  device cannot occupy memory trying to route garbage.

Messages and acknowledgements ride the mesh this way today; it is what makes
"delivered from the next room over" not depend on the router.

### Message delivery that survives interruption

Outbound messages go into a durable outbox first, then ship when a path
exists. If the network drops mid-send or the app is closed, nothing is lost:
the message is retried when the device is reachable again. Every message shows
its real state — queued, sent, delivered — rather than a hopeful checkmark.

### Calls without infrastructure

Voice and video calls use the same pairing as messages. The two devices
negotiate addresses through their existing encrypted channel and stream media
directly, sealed with keys derived from the call's own handshake. When a direct
path cannot be found, a third *paired* device can relay the call — still
sending nothing it can decrypt. Video rides the same transport as a small,
throttled JPEG stream so a call works even on a modest link.

### Notifications you can answer from

Messages wake the phone through a foreground service that keeps the mesh
listening quietly. Notifications are standard Android conversation
notifications: reply inline without opening the app, and the reply is sent by
the same encrypted path as any other message.

### Groups and files

Group messages fan out as individually encrypted copies, so every member
decrypts only what was addressed to them. Files and photos are chunked, sent
over the same encrypted links, and stored on-device only.

---

## What already works

- Encrypted identity, pairing, and session establishment (X3DH + Double
  Ratchet), with 166 unit tests covering the protocol, transport, delivery,
  relay, and call layers.
- Direct messaging with delivery states, a durable outbox, retries, typing
  indicators, reactions, and read receipts — all working with **no internet
  and no mobile data**, on a shared Wi-Fi or hotspot.
- **Multi-hop message relaying** through chains of paired devices when the
  destination is not on this network — sealed end to end, forwarded blind,
  loop-proofed by hop limit and dedup.
- One-to-one voice calls, with mute, speaker, call relay through a paired
  device, and a foreground call service.
- Video calls (camera frames sealed and chunked over the same call link).
- Conversation notifications with inline reply, and an onboarding screen that
  works without a PIN.

## What is still coming

- **Wi-Fi Direct transport** — device-to-device links with no router at all.
- **At-rest database encryption** (SQLCipher) so message bodies are encrypted
  on disk, not only in transit.
- **Group calling**, richer group management, and a signed release build.
- Device-verified testing of every path above, on real hardware.

---

## Honest limits

- Range is bounded by physics, not by servers: Nova reaches as far as its
  paths do — the network you share, the hotspot you make, or the chain of
  paired devices between you. Where a bigger range is needed, more paired
  devices make a bigger mesh; Nova improves the range it can reach rather than
  pretending a server could.
- Some networks isolate their clients from each other; Nova detects that and
  says so instead of hanging.
