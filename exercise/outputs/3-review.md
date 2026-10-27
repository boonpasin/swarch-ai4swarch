Agent: Claude Code subagent, model Claude Sonnet 5.5 (claude-sonnet-5-5)

# Review of ADR-0002 (RabbitMQ `OrderPlaced` event)

Evidence: `docs/adr/0002-publish-order-placed-events-to-rabbitmq.md` (the ADR), `docs/scenario.md`.

## 1. Quality-attribute scenarios

| # | Source · Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|
| S1 Performance | Customers · `PlaceOrder` | 11.11 peak, 200 orders/s, broker healthy **and** broker down | Order completes; event is published or buffered | p95 <= 500 ms in both states; 0 failed orders |
| S2 Availability / Correctness | MailFast outage (or consumer crash) · events keep arriving | Peak load | Orders succeed; loyalty still awarded once | Loyalty p99 < 60 s, exactly once; 0 lost `OrderPlaced` events |
| S3 Performance / Modifiability | MailFast p95 1.5 s, 0.5 % failures, 50/s limit · 200 emails/s produced | 11.11 peak | Email is rate-limited and retried, never sent over the limit | <= 50 requests/s at MailFast; every order emailed; delay stated and accepted by business |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Separate durable queue per action, with the email consumer capped at 45/s | A 558k email backlog cannot delay `loyalty` (S2). The 45/s cap keeps 5/s headroom under MailFast's 50/s limit. |
| Non-risk | Loyalty exactly-once through a unique constraint on order ID, with at-least-once delivery | Redelivery is safe. The effect is exactly once, as the scenario requires. |
| Non-risk | RabbitMQ over Kafka | Fits the team (two RabbitMQ users), Kubernetes, and 200 events/s. Replay is not a stated requirement. |
| Sensitivity point | The 100 ms publish-confirm timeout | It uses half of the roughly 200 ms spare budget (500 - 300). While the broker is down, every order pays this 100 ms, so p95 is approximately 400 ms, close to the limit. The "< 20 ms p95" target for a quorum-queue confirm is not evidenced. |
| Sensitivity point | Email limit (45/s) times the number of consumer replicas, plus retries | Rate limiting is described per consumer. Two replicas, or retries counted on top of the 45/s, exceed MailFast's 50/s. Not specified in the ADR. |
| Trade-off point | Email delay (about 3.4 h backlog on a 1 h peak) against availability and performance | The ADR gains availability and latency but pays in user-perceived quality. The business has not agreed to this delay (open question). |
| Trade-off point | In-memory retry buffer against "never fail an order" | Availability and latency are preserved, but correctness is weakened: events can be lost. |
| Risk | The loss window is argued away with "3-node quorum cluster", but that protects data inside the broker, not the in-memory buffer in a `checkoutservice` pod. The pods that hold events are the ones that restart or scale during a sale. | A lost event means loyalty points are never awarded and no email is sent. This breaks the driver "loyalty within 1 minute, exactly once" while being described as a small accepted risk. |
| Risk | Dead-lettering after 5 attempts during a MailFast outage | A queue with a 500k+ backlog would drain into the DLQ within minutes, so the backlog becomes dead letters. No replay procedure is given. Backoff and timeout values (MailFast p99 is 8 s) are not stated. |
| Risk | The email consumer "records each order ID it has sent", but no store is named | The ADR says there is no order database and treats storage as an open question, yet the email consumer needs durable dedup. The duplicate-email mitigation has no home. |
| Risk | Analytics: "separate reduced message **or** the consumer strips it" | Leaving it open undecided. If the consumer strips the field, the email address still travels through the analytics queue. Privacy is not settled. |
| Risk | Validation: a 30-minute test at 200/s | It never builds the hour-long backlog the ADR itself assumes. The chaos test passes on "0 orders fail" and does not measure lost events. |

## 3. Missing or weak

- **Options not considered:** a transactional outbox (even a small DB or file-backed store), which is the only option that closes the loss window; a lighter broker (NATS JetStream, Redis Streams); batching or a higher rate limit from MailFast as a complement to queueing.
- **Hybrid not considered:** loyalty is only needed within 1 minute, so a synchronous gRPC call with a timeout and an async fallback was not evaluated.
- **Unsupported claims:** the "< 20 ms p95" publish target; "residual risk accepted" without a number (expected lost events per sale); the 1 h peak is an assumption and drives the sizing of the backlog and the RabbitMQ disks.
- **Consequences not stated:** RabbitMQ disk and memory sizing; the choice of database for `loyaltyservice`; what happens to emails older than the business accepts (are they still sent?); the cost of retiring `emailservice` and the migration order.
- **Scenario fact left unaddressed:** 150 emails/s of excess is queued, but the ADR does not say whether MailFast's limit can be raised, which is the cheapest fix.

## 4. Verdict

**Accept with changes.** The core decision (RabbitMQ, one queue per action, idempotent loyalty) is sound. The failure paths are not.

Three most important changes:
1. Close or quantify the event-loss window. Either add a durable outbox, or state the expected loss and get the business to sign off, since loyalty depends on it. Drop the claim that the broker's 3-node cluster covers it.
2. Define the email path under stress: global (not per-replica) 45/s limit including retries, the durable dedup store, and DLQ and replay rules so a MailFast outage does not turn the backlog into dead letters. Get the business to agree the email delay.
3. Decide the analytics payload now (publish a separate event without the email address) and extend validation: a test with a 1 h backlog, a measured publish latency with the broker down, and a count of lost events in the chaos test.
