# ADR-001: Outbound-Only WebSocket Tunnel Between On-Prem and Cloud

**Status:** Accepted, in production since platform GA
**Deciders:** Platform architecture, with input from customer security/infosec teams during early design partner engagements

## Context

The platform's whole value proposition is bridging cloud-hosted consumer apps to on-prem
enterprise content systems (ECMs) — document repositories that live inside customer data
centers for compliance and data-residency reasons. Those repositories are never going to be
directly reachable from the internet.

We needed a way for the cloud control plane (API Gateway-fronted Gateway API) to dispatch work
to an on-prem agent (the Broker) and get results back, in something close to real time,
without asking every enterprise customer's infosec team to open an inbound port into their
network. For an enterprise sales motion, "please open a firewall rule for us" is close to a
non-starter — it adds a security review cycle to every customer onboarding and becomes a
support burden forever after (rule drift, VPN changes, IP allowlist churn).

## Options considered

1. **Inbound connection from cloud to on-prem Broker** (traditional client-server, cloud
   polls or pushes to an on-prem HTTP listener).
   - Rejected outright. Requires customers to expose an on-prem service to the internet, or
     stand up VPN/PrivateLink infrastructure per customer. Kills the "SaaS, no on-prem infra
     team required" pitch.

2. **Polling: Broker periodically polls cloud API Gateway for work.**
   - Broker initiates every connection, so it satisfies the no-inbound-ports constraint.
   - Simple to build and reason about.
   - Rejected as the primary mechanism because polling latency is bounded by poll interval —
     tight polling to get low latency means high request volume and cost at scale (hundreds
     of customer Brokers polling every few seconds, most of the time for nothing).

3. **Outbound persistent WebSocket tunnel: Broker opens and holds a long-lived WebSocket
   connection outbound to the Gateway API.**
   - Broker initiates the connection (satisfies the constraint), then the channel is
     bidirectional and low-latency for the life of the connection.
   - Cloud can push work to the Broker the moment it's available, no poll delay.
   - Added complexity: connection lifecycle management, reconnect/backoff, keepalive,
     multi-instance connection tracking (which Gateway pod owns which Broker's socket).

## Decision

We went with the outbound-only persistent WebSocket tunnel, using native ASP.NET Core
WebSockets on the cloud side rather than a managed WebSocket API Gateway product. The Broker
authenticates once via a bootstrap-token exchange, then upgrades to a long-lived connection and
maintains it with reconnect/keepalive logic. All actual dispatch (job requests, results) flows
over this channel; binary payloads explicitly do **not** (see [ADR-002](002-s3-presigned-binary-transfer.md)).

We chose to own the WebSocket implementation directly in the API host rather than adopt a
managed WebSocket gateway product, specifically so that connection-routing logic (which
Gateway instance is holding which Broker's socket) stayed in our control — this became
important for [ADR-004](004-dynamodb-connection-state.md), where we needed to track and query
that mapping across horizontally-scaled Gateway instances.

## Tradeoffs

**What we gave up:**
- Managed WebSocket infrastructure (e.g., a managed API gateway WebSocket product) would have
  handled connection scaling and routing for us. We took on that complexity ourselves in
  exchange for control over auth, message framing, and connection-state visibility.
- A persistent connection per customer Broker is a standing resource cost and a thing that can
  silently die (network blip, load balancer idle timeout) — we had to build explicit
  keepalive/reconnect and monitoring for "Broker appears connected but isn't responding,"
  which was a genuine operational headache in the first year.
- Every Gateway release that touches the WebSocket protocol is a coordinated release with every
  Broker version in the field — this is effectively a versioned wire protocol now, and we pay a
  compatibility tax for it.

**What we got:**
- Zero inbound firewall requirement for any customer — this removed a security-review blocker
  from the sales/onboarding cycle entirely.
- Low-latency, cloud-initiated dispatch without polling overhead.
- A single mechanism (not a grab bag of per-feature workarounds) that every future
  cloud-to-on-prem feature could build on.

## Outcome

This became a hard architectural constraint documented and enforced across the platform: any
new feature needing real-time cloud-to-on-prem communication reuses this tunnel rather than
inventing a new channel. It has held up in production at multi-region, multi-hundred-customer
scale. The main lesson from running it: the reconnect/backoff and connection-health-detection
logic needed far more investment than the happy-path dispatch code — that's where the
production incidents actually came from (stale connections that looked healthy, thundering-herd
reconnects after a Gateway deploy).

## What I'd reconsider today

If I were starting fresh at larger scale, I'd look harder at a managed WebSocket gateway
product up front, specifically to offload connection-state tracking — we ended up building a
DynamoDB-backed version of that ourselves (ADR-004) anyway, which is functionally similar to
what a managed service would give you, just self-built.
