# ADR-003: Lambda + SQS for Async Subscription Provisioning

**Status:** Accepted, in production

## Context

When a customer subscribes to the platform (a new "app subscription" event fired by the
upstream subscription-management system), a set of control-plane resources needs to be
provisioned for them before the platform is usable: tenant-scoped configuration, initial
records, and whatever setup a given app integration specifically requires. This provisioning
step is triggered by an external event (a subscription being created), not by a direct user
request — there's no HTTP caller sitting there waiting synchronously for a response.

The provisioning work itself is variable in duration and can fail transiently (downstream
dependency hiccups, throttling), and we needed it to be reliable — a dropped provisioning event
means a customer who paid for the product and can't use it, with no obvious signal to anyone
that something went wrong.

## Options considered

1. **Synchronous HTTP handler inside the main Gateway API** that reacts to a webhook/callback
   from the subscription system.
   - Simple, one fewer moving part. But couples provisioning reliability directly to Gateway
     API uptime and deploy cadence, and a slow or failing provisioning step risks tying up
     Gateway API request-handling capacity or, worse, silently dropping the event if the
     request times out with no retry.

2. **A long-running worker/consumer service** (containerized, always-on) subscribed to the
   event stream.
   - Reasonable, but for what is a relatively low, bursty volume of events (subscriptions
     aren't created at high steady-state rate), an always-on service is paying for idle compute
     most of the time, and it's one more service to deploy, patch, and keep healthy.

3. **Event-driven serverless: subscription events land on a queue; a Lambda function
   processes them.**
   - Pay only for actual invocations. Natural retry/backoff and dead-letter handling from the
     queue. Scales to zero when nothing's happening and scales out automatically under burst
     (e.g., a large customer's subscription creating many downstream app provisioning events at
     once).

## Decision

We built the provisioning flow as an SQS-triggered Lambda. Subscription-creation events (published
by the upstream subscription system as CloudEvents) land on an SQS queue; a Lambda function
processes each message, performs the provisioning work needed for that app/tenant, and replies
to a specified reply address on completion. Failed messages fall back to SQS's built-in
retry-with-backoff and eventually a dead-letter queue rather than being silently dropped.

We deliberately modeled the Lambda's setup similarly to our existing .NET API hosting patterns
(shared bootstrapping, DI, configuration) so that the team didn't need a fundamentally different
mental model or toolchain to work on it versus the rest of the codebase — this was a conscious
choice to keep the "serverless" pieces from becoming a separate, unfamiliar island that only one
person on the team could safely touch.

## Tradeoffs

**What we gave up:**
- Local development and debugging is a genuinely worse experience than a normal API — you're
  running against a Lambda test-harness/emulator rather than just running the service, and it's
  an extra tool in the onboarding checklist for new engineers.
- Cold starts are a real, if usually minor, latency cost on low-traffic paths — acceptable here
  because provisioning is inherently async and nobody's blocking on sub-second response, but
  it's not a pattern we'd default to for anything latency-sensitive.
- Debugging a failed provisioning run in production means correlating across queue message,
  Lambda invocation logs, and the reply — more moving parts to trace through than a single
  service's request log.

**What we got:**
- Provisioning failures don't get silently lost — SQS retry and dead-letter queue gave us a
  durable, inspectable "this didn't go through" state, which we didn't reliably have with the
  earlier synchronous approach.
- Genuinely near-zero idle cost, since subscription events are bursty and infrequent relative to
  the platform's other traffic.
- Decoupling provisioning from Gateway API's own release cadence and request-handling capacity —
  a slow provisioning run or a Gateway API deploy in progress no longer has any effect on the
  other.

## Outcome

This has run in production without needing rework since it shipped. The dead-letter queue has
been genuinely useful operationally — on the handful of occasions a downstream dependency the
provisioning step relies on had an outage, provisioning events queued up safely and drained
automatically once the dependency recovered, with zero data loss and no manual intervention.

## What I'd reconsider today

The event contract (CloudEvents-shaped payloads with a schema reference) has served us well and
I'd keep that. Where I'd push harder next time: standardizing observability across Lambda and
the rest of the .NET fleet from day one — we initially had a gap where Lambda invocations were
harder to correlate in traces than our containerized services, and closing that gap was
retrofit work rather than being designed in from the start.
