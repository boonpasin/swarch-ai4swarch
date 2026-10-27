Agent: Claude Code subagent, model Claude Sonnet 5.5 (claude-sonnet-5-5)

# ADR-0002: Publish an "order placed" event and let post-order actions subscribe to it

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Not evidenced in the provided files (see Open questions)

## Context and problem

Run `a` is zero-shot. This ADR is written from the requirement text alone; no repository file other than the template was read. Facts about the existing system below come only from the requirement.

After an order is placed, Online Boutique must:

1. Send the order confirmation email through **MailFast**, a new external email provider. This replaces the mock `emailservice` logic.
2. Award loyalty points through a new `loyaltyservice` (to be built).
3. Record an "order placed" analytics event for the marketing team.

Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

The problem: where does the knowledge of "what happens after an order" live, and how are these actions triggered? Today there are 3 actions, and 3 more are expected. Two involve a third party (MailFast) or a new service that does not exist yet, so they can fail or be slow independently of placing the order.

What is **not** known from the requirement: how the order flow is currently wired, whether a message broker already exists, volumes, latency targets, and whether any action (e.g. fraud scoring) must be able to block an order. See Open questions.

## Decision drivers

- Modifiability: adding the next post-order action (SMS, fraud scoring, recommendations update) should need little or no change to the order flow.
- Availability: a failing or slow MailFast, `loyaltyservice` or analytics sink must not prevent or delay a customer's order.
- Performance: the customer's checkout response time should not grow with every added action.
- Reliability of side effects: a placed order should not silently lose its confirmation email, points or analytics event.
- Cost / effort: the solution must be deliverable by the team (team skills and any broker already in use are unknown).
- Testability / operability: failures of individual actions must be visible and retryable.

## Considered options

1. **Direct synchronous calls from the order-placing service.** The service that completes the order calls MailFast (via an adapter), `loyaltyservice` and the analytics sink one after another.
2. **Event-driven fan-out through a message broker.** The order-placing service publishes one "order placed" event. The email adapter, `loyaltyservice` and the analytics recorder each subscribe independently. New actions are new subscribers.
3. **A dedicated post-order orchestrator service.** The order-placing service makes one call to a new orchestrator, which owns the list of post-order actions and calls each one (sync or async).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Direct synchronous calls | Simplest to build and debug; no new infrastructure; easy to reason about ordering. | Order flow must change for every new action (poor modifiability). Checkout latency and availability depend on MailFast and every other callee unless calls are made fire-and-forget. Retries and partial failure handling must be written inside the order flow. Coupling grows from 3 to 6+ dependencies. |
| 2. Event fan-out via broker | New actions are added by adding a subscriber, with no change to the publisher. Order completion is decoupled from MailFast and `loyaltyservice` failures. Broker gives buffering and retry. Each consumer scales and fails independently. | New infrastructure to run and operate (broker, or confirming that one exists). Eventual consistency: email, points and analytics happen slightly after the order. Needs idempotent consumers, duplicate handling and a dead-letter strategy. Harder end-to-end tracing. Fits poorly if an action must be able to reject an order. |
| 3. Dedicated orchestrator | Single place that knows the post-order workflow; the order flow makes one call and stays unchanged as actions grow. Central place for retries and monitoring. No broker required. | One more service to build and run. New actions still require changing the orchestrator (a smaller but real modifiability cost). It can become a bottleneck and a single point of failure, and tends to accumulate business logic. |

## Decision

Choose **Option 2: publish an "order placed" event to a message broker, and implement each post-order action as an independent subscriber.**

- The email adapter (calling MailFast), `loyaltyservice` and the analytics recorder each subscribe to the event.
- This best satisfies the top drivers. Modifiability: the expected SMS, fraud scoring and recommendations update become new subscribers. Availability and performance: the customer's order does not wait for, or fail because of, MailFast or `loyaltyservice`.
- Option 1 is rejected because every new action would change the order flow and add latency and failure modes to checkout. Option 3 is a reasonable fallback if the team cannot operate a broker (see Risks), but it keeps a central change point.
- Caveat: if fraud scoring turns out to need to **block** an order before it is confirmed, that action is not a post-order subscriber and should be decided separately (it would be a synchronous step in the order flow).

## Consequences

- **Positive:** Adding a post-order action does not change the publisher. (quality attribute: modifiability)
- **Positive:** MailFast outage or slowness, or a `loyaltyservice` outage, does not fail or slow the order; events wait in the broker and are retried. (quality attribute: availability, performance)
- **Positive:** The mock `emailservice` logic can be swapped for MailFast behind one subscriber without touching other actions. (quality attribute: modifiability)
- **Negative:** A message broker must be introduced or reused, deployed, monitored and secured. (quality attribute: cost, operability)
- **Negative:** Confirmation email, points and analytics are eventually consistent; a customer may briefly see an order without points. (quality attribute: consistency / user-perceived correctness)
- **Negative:** Debugging a single order across publisher and subscribers is harder without correlation IDs and tracing. (quality attribute: operability, testability)
- **Risks and mitigations:**
  - Event lost between order commit and publish: use a transactional outbox or equivalent at-least-once publish (feasibility depends on the order service's storage, not known).
  - Duplicate delivery causing double points or double email: make consumers idempotent, keyed by order ID.
  - Poison or repeatedly failing messages: retries with backoff and a dead-letter queue, with alerting.
  - MailFast rate limits, credentials and data-privacy terms: not known; handle in the email adapter and review before go-live.
  - Team cannot operate a broker: fall back to Option 3.

## Validation

- Fault-injection test: make MailFast and `loyaltyservice` unavailable and confirm orders still succeed and the email/points are delivered after recovery, with no duplicates.
- Measure checkout response time before and after; it should not increase with the number of subscribers.
- Add a dummy fourth subscriber (e.g. a stub SMS action) and confirm it needs no change to the publisher.
- Track the metric "events published vs. events processed per subscriber" and the dead-letter queue depth.
- Review this ADR when the first of SMS, fraud scoring or recommendations update is scheduled, and especially if fraud scoring must block orders.

## Open questions

- Who are the deciders (architect, team lead, marketing, security)?
- Is a message broker already part of Online Boutique, or must one be introduced? Not evidenced (no system files were read in this run).
- How does the order flow currently trigger `emailservice`, and which service would publish the event? Not evidenced.
- Expected order volume and the acceptable delay for the confirmation email, points and analytics event (no numbers given).
- Must fraud scoring be able to block or hold an order before confirmation?
- Does the analytics event need to be real-time, and what is its schema and destination for the marketing team?
- MailFast: API limits, availability SLA, cost, credentials handling and personal-data (privacy) requirements.
- Does `loyaltyservice` need its own data store, and what are the points rules (e.g. refunds and cancellations)?
- Team skills and the operational budget for running a broker.
- Should the event carry full order details or only an order ID (data-privacy and coupling trade-off)?
