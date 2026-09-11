# ADR-004: DynamoDB for WebSocket Connection and Event-Reply State

**Status:** Accepted, in production

## Context

The Gateway API runs as a horizontally scaled fleet of stateless pods, and each on-prem
Broker holds a persistent WebSocket connection (ADR-001) to exactly one Gateway pod at a time.
When a request comes in that needs to reach a specific customer's Broker, whichever pod handles
that request has to know which pod in the fleet is actually holding that Broker's socket —
sockets aren't shareable across processes.

A related problem showed up later with the events/pub-sub platform (ADR-005): request/reply
event interactions need to track in-flight request state (who published it, what responses have
come back, whether the lifecycle is still open) across a system that's also horizontally scaled
and where a given publisher's poll request could land on any pod.

Both problems are the same shape: **small, high-churn, per-connection or per-request pieces of
state that every pod in a stateless fleet needs to read and update, with low and predictable
latency.**

## Options considered

1. **In-memory state per pod, with gossip/broadcast to keep pods in sync.**
   - Rejected — building a correct gossip protocol for connection ownership is a distributed
     systems problem we didn't need to take on, and it doesn't survive pod restarts cleanly.

2. **Relational database (we already run PostgreSQL/Aurora for the platform's durable
   relational data).**
   - Would work functionally, but this state is high write-churn (every connect/disconnect,
     every keepalive, every event reply) and doesn't need relational integrity or complex
     queries — it's fundamentally key-value lookups ("which pod owns this Broker's connection,"
     "what's the state of this request ID"). Using the relational store for this shape of
     workload would add write load to infrastructure that's tuned for our actual relational
     data, for no relational benefit.

3. **A distributed cache (Redis/Valkey), which we also already run for session state
   elsewhere in the platform.**
   - A reasonable contender — fast, key-value shaped. We didn't choose it here mainly because
     we wanted this state to be durable, not just fast-access/ephemeral — a cache eviction or
     restart losing "which pod owns this connection" mid-operation was a worse failure mode than
     the latency cost of a managed durable key-value store.

4. **A managed, serverless key-value store (DynamoDB).**
   - Key-value shaped, scales automatically with connection/request volume, durable, and pairs
     naturally with the Lambda-based provisioning path (ADR-003) already using the AWS-native
     toolchain.

## Decision

We used DynamoDB for both connection-metadata tracking (which pod owns which Broker's socket,
used to route dispatch correctly across the fleet) and for events-platform request/reply
lifecycle state (request metadata, sequence-numbered replies, terminal/expired status).

## Tradeoffs

**What we gave up:**
- No relational queries or joins — anything we want to ask about this data has to be shaped
  around the access patterns we designed the keys for up front. A few times we wanted an
  ad-hoc "show me all open requests for tenant X older than N minutes"-style query for
  debugging, and had to either maintain a secondary index for it in advance or fall back to a
  scan, which is exactly the kind of thing a relational store would give you for free.
- Another data store type in the platform's operational surface (alongside Aurora Postgres and
  Redis/Valkey) — more to monitor, back up, and reason about consistency for.
- Item-size and throughput-partitioning considerations had to be designed for up front (hot
  partition risk if one very active tenant dominates a partition key) — this needed real
  thought at design time, not something you can ignore and fix later without a migration.

**What we got:**
- Effectively unbounded horizontal scale for connection/request volume without us managing
  capacity — this mattered directly as the platform went multi-region and added customers
  faster than our original connection-tracking assumptions anticipated.
- Durable state that survives pod restarts and deploys cleanly, so a Gateway API rolling deploy
  doesn't require any special-cased "drain connections carefully or lose state" handling.
- Low, predictable per-item read/write latency at the scale we operate at, which matters because
  this lookup sits directly in the hot path of every dispatched job and every event poll.

## Outcome

This has scaled cleanly through multi-region expansion (we run separate regional deployments,
each with its own DynamoDB tables, rather than a single global table — a deliberate simplicity
choice to avoid the operational complexity of cross-region replication for state that's
inherently regional anyway). The main real-world pain point has been the loss of ad-hoc
queryability during incident debugging, which we've partially mitigated with targeted secondary
indexes added after we learned what questions we actually needed to ask in production.

## What I'd reconsider today

I'd invest more upfront in the access-pattern exercise (writing out every query we'll ever need
to make against this table before deciding on keys/indexes) rather than adding secondary indexes
reactively after hitting a debugging wall. That's a case where the "NoSQL forces you to know
your access patterns up front" advice is correct and we didn't fully take it seriously enough at
design time.
