# ADR-002: S3 Presigned URLs for Binary Content, Not the WebSocket Tunnel

**Status:** Accepted, in production
**Deciders:** Platform architecture; triggered by production stability issues after initial GA

## Context

The platform moves documents — sometimes large ones (tens to low-hundreds of MB: scanned
records, multi-page case files, renditions) — between consumers, the cloud control plane, and
the on-prem Broker. Early on, it was tempting to just send binary payloads as WebSocket frames
over the same tunnel described in [ADR-001](001-outbound-only-tunnel.md), since the channel
already existed and was authenticated.

That's roughly what an early iteration did, and it caused real problems: a single large file
transfer could saturate a Broker's WebSocket connection for the duration of the transfer,
during which no other job dispatch or result could get through that same channel. At scale,
this meant one customer's large-document workflow could visibly degrade responsiveness for
every other concurrent job routed through the same connection.

## Options considered

1. **Keep binary content on the WebSocket tunnel**, but add chunking, backpressure, and
   prioritization so large transfers don't starve smaller messages.
   - Rejected as the primary fix. It's solvable in principle but adds significant complexity
     to a channel we wanted to keep simple and reliable for control-plane traffic. It also
     doesn't fix the fundamental issue: one physical connection is now doing two jobs with very
     different throughput/latency profiles (control messages vs. bulk transfer).

2. **Separate dedicated connection per transfer** (e.g., open a second WebSocket or raw TCP
   stream just for binary data).
   - Still constrained by the "no inbound to on-prem" rule for the download direction, and adds
     another connection type to manage, secure, and monitor. Doesn't obviously simplify
     anything relative to option 3.

3. **Move binary content out of the tunnel entirely: use object storage with presigned URLs.**
   Upload path: consumer/Gateway writes to object storage; Broker is handed a presigned GET URL
   and downloads directly. Download path: Broker uploads to object storage; consumer is handed
   a presigned GET URL and downloads directly. The WebSocket tunnel carries only the
   presigned URL and metadata, never the bytes.

## Decision

We adopted the presigned-URL pattern (S3, in our case) as a hard platform rule: **binary
content must not travel over the WebSocket tunnel.** This was later formalized as a documented
architectural constraint that any new feature involving file transfer has to follow — it's not
a per-feature judgment call anymore.

## Tradeoffs

**What we gave up:**
- An extra network hop and a dependency on object storage availability for any file-bearing
  workflow — a transfer now has more moving parts (get presigned URL → upload/download from
  storage → confirm) than a single WebSocket send.
- Presigned URL expiry and permission scoping had to be gotten right — too long an expiry is a
  security exposure (a leaked URL is a leaked document, no auth required to use it within its
  window), too short and legitimate large transfers over slow customer links fail.
- On-prem Brokers now need outbound HTTPS access to the object storage endpoint specifically
  (in addition to the WebSocket tunnel target), which is one more thing to document in customer
  network-egress requirements, though still fully outbound and generally already allowed.

**What we got:**
- The control-plane WebSocket connection stays lightweight and low-latency regardless of file
  size or count of concurrent transfers — a 200MB document transfer for one customer has zero
  effect on job dispatch responsiveness for others.
- Transfer throughput scales independently of the tunnel — object storage handles concurrent
  large transfers far better than we'd ever get multiplexing them over a single per-Broker
  socket.
- Encryption at rest (KMS-backed) and standard object storage lifecycle/retention policies come
  for free instead of us building bespoke storage handling into the tunnel protocol.

## Outcome

This fixed the "one customer's big file degrades everyone" problem outright, and became one of
the platform's non-negotiable constraints — codified in architectural guidance so that every
future feature touching binary content defaults to this pattern rather than re-litigating it.
It shipped alongside a broader deployment-model modernization (containerized Broker, hardened
base image) as part of the same release, which made it easier to land as a coordinated change
across every customer's Broker fleet at once.

## What I'd reconsider today

Presigned URL TTL tuning was trial-and-error in practice — we initially set it too
conservatively short and saw failures from customers on slower egress links, then widened it.
I'd bake in adaptive expiry (based on payload size / historical transfer time for that Broker)
from day one instead of a fixed global TTL.
