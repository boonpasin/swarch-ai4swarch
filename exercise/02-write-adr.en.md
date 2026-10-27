**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No. It had none, and correctly listed "volumes, latency targets, MailFast limits" under Open questions instead of inventing them. | Yes, all of them: 500 ms p95, 200 orders/s, MailFast 1.5 s / 8 s / 0.5 % / 50/s, 1 minute, team skills, self-hosted preference. It also derived the 150 emails/s excess. |
| Options considered (list them) | 1. Direct sync calls 2. Broker fan-out (chosen, broker unnamed) 3. Post-order orchestrator service | 1. Sync gRPC 2. Fire-and-forget goroutines 3. RabbitMQ + consumers (chosen) 4. Managed cloud messaging. Option 4 is a straw man (the scenario already rules it out), and self-hosted Kafka/NATS is missing. |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Generically: "idempotent consumers keyed by order ID". The rate limit isn't mentioned (it didn't know it). | Loyalty: idempotent on order ID, but no store or mechanism named. Rate limit: consumer "capped below 50/s". **But** it doesn't say each consumer gets its own queue, so the email backlog could delay loyalty past 1 minute. Its "best-effort publish, failure only logged" quietly breaks exactly-once (lost event = no points). |
| Anything invented (a number, tool or fact not in the scenario)? | Nothing factual. It added an orchestrator option and a fraud-blocking caveat, both reasonable. | No invented numbers. Minor: it says "`emptyUserCart`" runs before email (true per `main.go`) and suggests the `loadgenerator` for load testing (not in its context, though it does exist). Its claim that Option 4 has "no team experience evidenced" is a filler con. |

**Verdict:** b is far better (concrete, uses the numbers, names RabbitMQ to match the team), confirming T3: context closes most of the gap. a's value is its honest Open questions list and the orchestrator option, which b dropped.

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Publish failure: from "best-effort, only logged" to publisher confirms + 100 ms timeout + background retry buffer + failure metric, with the remaining loss window explicitly accepted and an outbox deferred | The AI's version silently loses events, so a customer would get no points. That breaks the exactly-once requirement while claiming to meet it. Online Boutique has no order DB, so an outbox isn't free; I made the trade-off visible instead of hiding it |
| 2 | One durable quorum queue per consumer on a topic exchange | Without separate queues, a 11.11 email backlog would delay loyalty past the 1-minute limit. Quorum queues on 3 nodes address the broker being a single point of failure |
| 3 | Exactly-once made concrete: `loyaltyservice` DB transaction + unique constraint on order ID | "Idempotent on order ID" alone isn't a mechanism; the board will ask how |
| 4 | Email consumer capped at 45/s (not just "below 50"), and the backlog worked out: approximately 558,000 emails / approximately 3.4 h to clear if the peak lasts 1 h (assumption marked) | Makes the cost of the decision visible to the product owner; "email delay grows" was too vague |
| 5 | Added self-hosted Kafka as a real alternative (runner-up) | The AI's Option 4 (managed cloud) was a straw man the scenario already excluded; ADRs need genuinely different options |
| 6 | Analytics consumer gets no email address | The AI's event put the user's email on every queue, including marketing analytics (privacy) |
| 7 | Validation given pass/fail criteria (30 min at 200/s, ≤ 50 req/s to MailFast, p99 points < 60 s) | The AI's checks had no pass threshold |
