# ADR-005: Kafka + SQS + DynamoDB for the Partner-Facing Events Platform

**Status:** Accepted, in production

## Context

Partners and consumer systems integrating with the platform needed a way to publish events (for
example, "a document changed") and either fire-and-forget them to interested subscribers, or
publish a request and asynchronously poll for one or more replies (request/reply). This had to
work across the same fundamental boundary as everything else here: publishers and subscribers
are external systems and on-prem components, not trusted internal services, and the platform
sits in the middle enforcing schema, ownership, and access control on every event.

Requirements that shaped this:
- Multiple subscribers might care about one event (classic pub/sub fan-out).
- Request/reply needed a durable, pollable lifecycle — a publisher fires a request, gets a
  request ID back immediately, and polls for responses over time (sometimes minutes later),
  rather than holding a connection open.
- Delivery had to be reliable enough for an enterprise SLA — an event silently dropped is a
  partner-visible failure, not an internal inconvenience.
- The publishing surface (via a Broker-hosted plugin) needed to accept an event and return
  quickly (202 Accepted) rather than block on full downstream delivery.

## Options considered

1. **Single queue (SQS) for everything — publish to a queue, subscribers each poll their own
   queue.**
   - Works fine for point-to-point delivery, but fan-out to multiple subscribers from one queue
     needs one queue per subscriber wired up by hand, and it doesn't give us a durable, replayable
     event log if we need to add a new subscriber after the fact or replay recent history —
     which we already anticipated needing (and which later shipped as "message replay for failed
     event deliveries").

2. **A managed pub/sub notification service (e.g., topic/subscription fan-out) alone,
   without a durable log.**
   - Simpler to operate than running a log-based broker, but doesn't give consumers replay or a
     durable retained history the way a log does — once delivered/expired, it's gone. Given we
     knew we'd need request/reply lifecycle tracking and eventual replay, this was too thin on
     its own.

3. **Kafka as the durable event backbone, with SQS as the per-subscriber delivery mechanism,
   and DynamoDB for request/reply lifecycle metadata.**
   - Kafka gives us a durable, replayable, ordered log that many subscribers can consume from
     independently without publisher-side fan-out logic.
   - SQS in front of each subscriber gives us the reliable, individually-retryable delivery
     semantics (dead-letter handling, backoff) we already trusted from the provisioning flow
     (ADR-003).
   - DynamoDB tracks request/reply lifecycle state (ADR-004) since that's inherently key-value
     shaped, high-churn, per-request state.

## Decision

We built the events platform as: publisher → plugin validates against the event schema and
persists to a local outbox → 202 Accepted returned immediately → background worker dispatches
the event to the Gateway, which publishes it onto Kafka → subscribers consume via their own SQS
queue fed from Kafka → for request/reply, response metadata and replies are written to DynamoDB,
keyed and sequence-numbered so a publisher can poll and get replies in order, and clean up the
lifecycle explicitly when done.

The publish endpoint deliberately returns 202 immediately rather than waiting on end-to-end
delivery — the caller gets a fast, predictable response, and delivery reliability is the
platform's problem to solve asynchronously, not something we push onto the publisher's request
latency budget.

## Tradeoffs

**What we gave up:**
- This is the most operationally complex piece of infrastructure in the whole platform — three
  distinct systems (Kafka, SQS, DynamoDB) in one feature's critical path, each with its own
  failure modes, each needing its own monitoring and on-call runbook.
- End-to-end tracing across "publisher plugin → outbox → dispatch worker → Kafka → subscriber
  SQS → subscriber" is genuinely hard to reason about compared to a synchronous call chain —
  when a partner asks "where's my event," the answer requires correlating state across four or
  five systems.
- Running Kafka (even managed) is a heavier operational commitment than the queue-only
  alternative — partition management, consumer group health, and retention tuning are all real
  ongoing work, not "set it and forget it."

**What we got:**
- True fan-out without publisher-side complexity — a new subscriber can be added without the
  publisher or the platform's publish path changing at all.
- A durable, replayable event history, which became directly useful later when we needed to add
  message replay for failed deliveries — that capability came almost for free because the log was
  already there, rather than being a retrofit onto a queue-only design that discards messages
  after delivery.
- Clear separation of concerns: Kafka is the source of truth and fan-out mechanism, SQS is the
  reliable per-subscriber delivery primitive, DynamoDB is fast per-request lookup state. Each
  piece is doing the thing it's actually good at rather than one system straining to do all three
  jobs.

## Outcome

This shipped as the platform's real-time partner-notification capability and was one of the
larger deliveries of the period. Within a quarter of GA we added enterprise access controls,
audit logging, pagination, and message replay on top of this foundation — all of which were
straightforward additions specifically because the underlying log-based architecture already
supported them, rather than requiring a redesign.

## What I'd reconsider today

Given how much operational complexity this added, I'd want to be more disciplined about
validating, before committing to Kafka specifically, whether a simpler managed
pub/sub-plus-replay-log offering could have met the replay requirement with less operational
surface area. We made the Kafka call partly because the team already had Kafka operational
experience elsewhere in the org — a legitimate factor, but one I'd want to weigh explicitly
against the alternative next time rather than let prior familiarity be the deciding vote.
